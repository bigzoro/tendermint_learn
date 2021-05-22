# 概述
- state是共识引擎最新提交区块的一个简单描述，它包含了验证一个区块所有的必要信息，例如验证者集合、共识参数。
- state并不会在网络中进行传播。
- 对state进行操作时，应该使用state.Copy() 或者updateState()。
# state
```go
type State struct {
	// 共识版本
	Version Version

	// ChainID和InitialHeight的默认值都在创世块配置文件中进行配置。
	// 并且在整个链中都不应该变化
	ChainID       string
	InitialHeight int64 // should be 1, not 0, when starting from height 1

	// LastBlockHeight=0 at genesis (ie. block(H=0) does not exist)
	// 上一个区块的高度
	LastBlockHeight int64
	// 上一个区块的ID
	LastBlockID     types.BlockID
	// 上一个区块的时间
	LastBlockTime   time.Time

	// LastValidators is used to validate block.LastCommit.
	// Validators are persisted to the database separately every time they change,
	// so we can query for historical validator sets.
	// Note that if s.LastBlockHeight causes a valset change,
	// we set s.LastHeightValidatorsChanged = s.LastBlockHeight + 1 + 1
	// Extra +1 due to nextValSet delay.
	NextValidators              *types.ValidatorSet
	// 代表当前验证者集合
	Validators                  *types.ValidatorSet
	// 上一个区块验证者集合
	LastValidators              *types.ValidatorSet
	LastHeightValidatorsChanged int64

	// 共识参数的设置, 主要是一个区块大小、一个交易大小、区块每个部分的大小
	ConsensusParams                  types.ConsensusParams
	LastHeightConsensusParamsChanged int64

	// Merkle root of the results from executing prev block
	LastResultsHash []byte

	// the latest AppHash we've received from calling abci.Commit()
	AppHash []byte
}
```
# execution
execution处理区块的提交和状态的更新，它暴露了ApplyBlock()去验证和处理区块。然后提交和更新交易池，最后保存区块的state。
下面我们就看一下ApplyBlock。
## ApplyBlock
它是唯一需要从这个包外部调用来处理和提交整个区块的函数。ApplyBlock的主要功能有：
1. 根据当前状态和区块内容来验证当前区块是否符合要求
2. 提交区块内容到ABCI的应用层, 返回应用层的回应
3. 根据当前区块的信息，ABCI回应的内容，生成下一个State
4. 再次调用ABCI的Commit返回当前APPHASH
5. 持久化此处状态同时返回下一次状态的内容
```go
func (blockExec *BlockExecutor) ApplyBlock(
	state State, blockID types.BlockID, block *types.Block,
) (State, int64, error) {

	// 对区块进行详细验证
	if err := validateBlock(state, block); err != nil {
		return state, 0, ErrInvalidBlock(err)
	}

	startTime := time.Now().UnixNano()

	// 将区块内容提交给ABCI应用层、返回ABCI返回的结果
	abciResponses, err := execBlockOnProxyApp(
		blockExec.logger, blockExec.proxyApp, block, blockExec.store, state.InitialHeight,
	)
	endTime := time.Now().UnixNano()
	blockExec.metrics.BlockProcessingTime.Observe(float64(endTime-startTime) / 1000000)
	if err != nil {
		return state, 0, ErrProxyAppConn(err)
	}

	fail.Fail() // XXX

	// 在提交前保存结果
	if err := blockExec.store.SaveABCIResponses(block.Height, abciResponses); err != nil {
		return state, 0, err
	}

	fail.Fail() // XXX

	// validate the validator updates and convert to tendermint types
	abciValUpdates := abciResponses.EndBlock.ValidatorUpdates
	err = validateValidatorUpdates(abciValUpdates, state.ConsensusParams.Validator)
	if err != nil {
		return state, 0, fmt.Errorf("error in validator updates: %v", err)
	}

	validatorUpdates, err := types.PB2TM.ValidatorUpdates(abciValUpdates)
	if err != nil {
		return state, 0, err
	}
	if len(validatorUpdates) > 0 {
		blockExec.logger.Debug("updates to validators", "updates", types.ValidatorListString(validatorUpdates))
	}

	// 通过block和客户端的响应来更新state
	state, err = updateState(state, blockID, &block.Header, abciResponses, validatorUpdates)
	if err != nil {
		return state, 0, fmt.Errorf("commit failed for application: %v", err)
	}

	// 调用ABCI的commit函数，返回APPHash，同时更新内存池中的交易
	appHash, retainHeight, err := blockExec.Commit(state, block, abciResponses.DeliverTxs)
	if err != nil {
		return state, 0, fmt.Errorf("commit failed for application: %v", err)
	}

	// 更新证据池的内容
	blockExec.evpool.Update(state, block.Evidence.Evidence)

	fail.Fail() // XXX

	// Update the app hash and save the state.
	state.AppHash = appHash
	// 将此次状态的内容持久化保存
	if err := blockExec.store.Save(state); err != nil {
		return state, 0, err
	}

	fail.Fail() // XXX

	// Events are fired after everything else.
	// NOTE: if we crash between Commit and Save, events wont be fired during replay
	fireEvents(blockExec.logger, blockExec.eventBus, block, abciResponses, validatorUpdates)

	return state, retainHeight, nil
}
```
下面看一些ApplyBlock中调用的函数
## validateBlock
验证区块
```go
func validateBlock(state State, block *types.Block) error {

	// 先对区块数据结构进行验证 看看是否参数都已经正确
	if err := block.ValidateBasic(); err != nil {
		return err
	}

	// 验证基本的信息
	if block.Version.App != state.Version.Consensus.App ||
		block.Version.Block != state.Version.Consensus.Block {
		return fmt.Errorf("wrong Block.Header.Version. Expected %v, got %v",
			state.Version.Consensus,
			block.Version,
		)
	}
	// 验证链ID 高度 上一个区块ID 区块交易的数量
	if block.ChainID != state.ChainID {
		return fmt.Errorf("wrong Block.Header.ChainID. Expected %v, got %v",
			state.ChainID,
			block.ChainID,
		)
	}
	if state.LastBlockHeight == 0 && block.Height != state.InitialHeight {
		return fmt.Errorf("wrong Block.Header.Height. Expected %v for initial block, got %v",
			block.Height, state.InitialHeight)
	}
	if state.LastBlockHeight > 0 && block.Height != state.LastBlockHeight+1 {
		return fmt.Errorf("wrong Block.Header.Height. Expected %v, got %v",
			state.LastBlockHeight+1,
			block.Height,
		)
	}
	// Validate prev block info.
	if !block.LastBlockID.Equals(state.LastBlockID) {
		return fmt.Errorf("wrong Block.Header.LastBlockID.  Expected %v, got %v",
			state.LastBlockID,
			block.LastBlockID,
		)
	}

	// Validate app info
	if !bytes.Equal(block.AppHash, state.AppHash) {
		return fmt.Errorf("wrong Block.Header.AppHash.  Expected %X, got %v",
			state.AppHash,
			block.AppHash,
		)
	}
	hashCP := state.ConsensusParams.HashConsensusParams()
	if !bytes.Equal(block.ConsensusHash, hashCP) {
		return fmt.Errorf("wrong Block.Header.ConsensusHash.  Expected %X, got %v",
			hashCP,
			block.ConsensusHash,
		)
	}
	if !bytes.Equal(block.LastResultsHash, state.LastResultsHash) {
		return fmt.Errorf("wrong Block.Header.LastResultsHash.  Expected %X, got %v",
			state.LastResultsHash,
			block.LastResultsHash,
		)
	}
	// LastValidators 表示上次的所有验证者合集
	if !bytes.Equal(block.ValidatorsHash, state.Validators.Hash()) {
		return fmt.Errorf("wrong Block.Header.ValidatorsHash.  Expected %X, got %v",
			state.Validators.Hash(),
			block.ValidatorsHash,
		)
	}
	if !bytes.Equal(block.NextValidatorsHash, state.NextValidators.Hash()) {
		return fmt.Errorf("wrong Block.Header.NextValidatorsHash.  Expected %X, got %v",
			state.NextValidators.Hash(),
			block.NextValidatorsHash,
		)
	}

	// Validate block LastCommit.
	if block.Height == state.InitialHeight {
		if len(block.LastCommit.Signatures) != 0 {
			return errors.New("initial block can't have LastCommit signatures")
		}
	} else {
		// 注意这个地方 我们是根据此次提交的区块信息 来验证上一个块的内容
		// 迭代上一个区块保存的所有验证者 确保每一个验证者签名正确
		// 最后确认所有的有效的区块验证者的投票数要大于整个票数的2/3
		if err := state.LastValidators.VerifyCommit(
			state.ChainID, state.LastBlockID, block.Height-1, block.LastCommit); err != nil {
			return err
		}
	}

	// NOTE: We can't actually verify it's the right proposer because we don't
	// know what round the block was first proposed. So just check that it's
	// a legit address and a known validator.
	// The length is checked in ValidateBasic above.
	if !state.Validators.HasAddress(block.ProposerAddress) {
		return fmt.Errorf("block.Header.ProposerAddress %X is not a validator",
			block.ProposerAddress,
		)
	}

	// Validate block Time
	switch {
	case block.Height > state.InitialHeight:
		if !block.Time.After(state.LastBlockTime) {
			return fmt.Errorf("block time %v not greater than last block time %v",
				block.Time,
				state.LastBlockTime,
			)
		}
		medianTime := MedianTime(block.LastCommit, state.LastValidators)
		if !block.Time.Equal(medianTime) {
			return fmt.Errorf("invalid block time. Expected %v, got %v",
				medianTime,
				block.Time,
			)
		}

	case block.Height == state.InitialHeight:
		genesisTime := state.LastBlockTime
		if !block.Time.Equal(genesisTime) {
			return fmt.Errorf("block time %v is not equal to genesis time %v",
				block.Time,
				genesisTime,
			)
		}

	default:
		return fmt.Errorf("block height %v lower than initial height %v",
			block.Height, state.InitialHeight)
	}

	// Check evidence doesn't exceed the limit amount of bytes.
	if max, got := state.ConsensusParams.Evidence.MaxBytes, block.Evidence.ByteSize(); got > max {
		return types.NewErrEvidenceOverflow(max, got)
	}

	return nil
}
```

## execBlockOnProxyApp
execBlockOnProxyApp函数是向ABCI提交信息, 它将返回一系列交易和Validator的集合
```go
func execBlockOnProxyApp(
	logger log.Logger,
	proxyAppConn proxy.AppConnConsensus,
	block *types.Block,
	store Store,
	initialHeight int64,
) (*tmstate.ABCIResponses, error) {
	var validTxs, invalidTxs = 0, 0

	txIndex := 0
	// 首先构建Response
	abciResponses := new(tmstate.ABCIResponses)
	dtxs := make([]*abci.ResponseDeliverTx, len(block.Txs))
	abciResponses.DeliverTxs = dtxs

	// 回调函数，在提交每一个交易给ABCI之后 然后在调用此函数
	// 这个回调只是统计了哪些交易在应用层被任务是无效的交易
	// 从这里我们也可以看出来 应用层无论决定提交的交易是否有效 tendermint都会将其打包到区块链中
	proxyCb := func(req *abci.Request, res *abci.Response) {
		if r, ok := res.Value.(*abci.Response_DeliverTx); ok {
			// TODO: make use of res.Log
			// TODO: make use of this info
			// Blocks may include invalid txs.
			txRes := r.DeliverTx
			if txRes.Code == abci.CodeTypeOK {
				validTxs++
			} else {
				logger.Debug("invalid tx", "code", txRes.Code, "log", txRes.Log)
				invalidTxs++
			}

			abciResponses.DeliverTxs[txIndex] = txRes
			txIndex++
		}
	}
	proxyAppConn.SetResponseCallback(proxyCb)

	// 从区块中加载出整个验证者和错误验证者
	commitInfo := getBeginBlockValidatorInfo(block, store, initialHeight)
	byzVals := make([]abci.Evidence, 0)
	for _, evidence := range block.Evidence.Evidence {
		byzVals = append(byzVals, evidence.ABCI()...)
	}

	ctx := context.Background()

	// Begin block
	var err error
	pbh := block.Header.ToProto()
	if pbh == nil {
		return nil, errors.New("nil header")
	}

	// 开始调用ABCI的BeginBlock 同时向其提交区块hash 区块头信息 上一个区块的验证者  出错的验证者
	abciResponses.BeginBlock, err = proxyAppConn.BeginBlockSync(
		ctx,
		abci.RequestBeginBlock{
			Hash:                block.Hash(),
			Header:              *pbh,
			LastCommitInfo:      commitInfo,
			ByzantineValidators: byzVals,
		},
	)
	if err != nil {
		logger.Error("error in proxyAppConn.BeginBlock", "err", err)
		return nil, err
	}

	// 迭代提交每一个交易给应用层
	for _, tx := range block.Txs {
		_, err = proxyAppConn.DeliverTxAsync(ctx, abci.RequestDeliverTx{Tx: tx})
		if err != nil {
			return nil, err
		}
	}

	// 通知ABCI应用层此次区块已经提交完毕了  注意这个步骤是可以更新验证者的 更新的验证者也就是下一个区块的所有验证者
	abciResponses.EndBlock, err = proxyAppConn.EndBlockSync(ctx, abci.RequestEndBlock{Height: block.Height})
	if err != nil {
		logger.Error("error in proxyAppConn.EndBlock", "err", err)
		return nil, err
	}

	logger.Info("executed block", "height", block.Height, "num_valid_txs", validTxs, "num_invalid_txs", invalidTxs)
	return abciResponses, nil
}
```
## updateState
与应用层交互过后，就需要更新state了
```go
func updateState(
	state State,
	blockID types.BlockID,
	header *types.Header,
	abciResponses *tmstate.ABCIResponses,
	validatorUpdates []*types.Validator,
) (State, error) {

	// Copy the valset so we can apply changes from EndBlock
	// and update s.LastValidators and s.Validators.
	nValSet := state.NextValidators.Copy()

	// 根据abciResponses返回的验证者来更新当前的验证者集合
	// 更新原则是这样：
	// 如果当前不存在则直接加入一个验证者
	// 如果当前存在并且投票权为0则删除
	// 如果当前存在其投票权不为0则更新
	lastHeightValsChanged := state.LastHeightValidatorsChanged
	if len(validatorUpdates) > 0 {
		err := nValSet.UpdateWithChangeSet(validatorUpdates)
		if err != nil {
			return state, fmt.Errorf("error changing validator set: %v", err)
		}
		// Change results from this height but only applies to the next next height.
		lastHeightValsChanged = header.Height + 1 + 1
	}

	// Update validator proposer priority and set state variables.
	nValSet.IncrementProposerPriority(1)

	// 根据返回结果更新一下共识参数
	nextParams := state.ConsensusParams
	lastHeightParamsChanged := state.LastHeightConsensusParamsChanged
	if abciResponses.EndBlock.ConsensusParamUpdates != nil {
		// NOTE: must not mutate s.ConsensusParams
		nextParams = state.ConsensusParams.UpdateConsensusParams(abciResponses.EndBlock.ConsensusParamUpdates)
		err := nextParams.ValidateConsensusParams()
		if err != nil {
			return state, fmt.Errorf("error updating consensus params: %v", err)
		}

		state.Version.Consensus.App = nextParams.Version.AppVersion

		// Change results from this height but only applies to the next height.
		lastHeightParamsChanged = header.Height + 1
	}

	nextVersion := state.Version

	//返回此次区块被验证成功之后的State 此State也就是为了验证下一个区块
	//注意APPHASH还没有更新 因为还有一步没有做
	return State{
		Version:                          nextVersion,
		ChainID:                          state.ChainID,
		InitialHeight:                    state.InitialHeight,
		LastBlockHeight:                  header.Height,
		LastBlockID:                      blockID,
		LastBlockTime:                    header.Time,
		NextValidators:                   nValSet,
		Validators:                       state.NextValidators.Copy(),
		LastValidators:                   state.Validators.Copy(),
		LastHeightValidatorsChanged:      lastHeightValsChanged,
		ConsensusParams:                  nextParams,
		LastHeightConsensusParamsChanged: lastHeightParamsChanged,
		LastResultsHash:                  ABCIResponsesResultsHash(abciResponses),
		AppHash:                          nil,
	}, nil
}
```
## Save
将state进行持久化
```go
func (store dbStore) save(state State, key []byte) error {
	batch := store.db.NewBatch()
	defer batch.Close()

	nextHeight := state.LastBlockHeight + 1
	// If first block, save validators for the block.
	if nextHeight == 1 {
		nextHeight = state.InitialHeight
		// This extra logic due to Tendermint validator set changes being delayed 1 block.
		// It may get overwritten due to InitChain validator updates.
		if err := store.saveValidatorsInfo(nextHeight, nextHeight, state.Validators, batch); err != nil {
			return err
		}
	}
	// 保存Validator
	err := store.saveValidatorsInfo(nextHeight+1, state.LastHeightValidatorsChanged, state.NextValidators, batch)
	if err != nil {
		return err
	}

	// 保存共识参数
	if err := store.saveConsensusParamsInfo(nextHeight,
		state.LastHeightConsensusParamsChanged, state.ConsensusParams, batch); err != nil {
		return err
	}

	// 设置批处理
	if err := batch.Set(key, state.Bytes()); err != nil {
		return err
	}

	return batch.WriteSync()
}
```
# 最后
至此，Tendermint state源码就分析完了。由于逻辑不是太复杂，所以就直接引用很多参考的文章。如果有错误之处，希望路过的大佬能够指点指点。最后推荐一位大佬的公众号，欢迎关注哦：**区块链技术栈**

另外这个地址上还有很多区块链学习资料：[https://github.com/mindcarver/blockchain_guide](https://github.com/mindcarver/blockchain_guide)

参考文章：https://gitee.com/wupeaking/tendermint_code_analysis/blob/master/state%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md