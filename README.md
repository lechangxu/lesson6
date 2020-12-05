# lesson6

1、pallets/poe/src/lib.rs **********************************************************************************************************************************

#![cfg_attr(not(feature = "std"), no_std)]

/// Edit this file to define custom logic or remove it if it is not needed.
/// Learn more about FRAME and the core library of Substrate FRAME pallets:
/// https://substrate.dev/docs/en/knowledgebase/runtime/frame

use frame_support::{
	decl_module, decl_storage, decl_event, 
	ensure, decl_error, dispatch, traits::Get};
use frame_system::ensure_signed;
use sp_std::prelude::*;  // 使用Vec

#[cfg(test)]
mod mock;

#[cfg(test)]
mod tests;

/// Configure the pallet by specifying the parameters and types on which it depends.
pub trait Trait: frame_system::Trait {
	/// Because this pallet emits events, it depends on the runtime's definition of an event.
	type Event: From<Event<Self>> + Into<<Self as frame_system::Trait>::Event>;
}

// The pallet's runtime storage items.
// https://substrate.dev/docs/en/knowledgebase/runtime/storage
decl_storage! {
	// A unique name is used to ensure that the pallet's storage items are isolated.
	// This name may be updated, but each pallet in the runtime must use a unique name.
	// ---------------------------------vvvvvvvvvvvvvv
	trait Store for Module<T: Trait> as TemplateModule {

		Proofs get(fn proofs): map hasher(blake2_128_concat) Vec<u8> => (T::AccountId, T::BlockNumber)
		
	}
}

// Pallets use events to inform users when important changes are made.
// https://substrate.dev/docs/en/knowledgebase/runtime/events
decl_event!(
	pub enum Event<T> where AccountId = <T as frame_system::Trait>::AccountId {
		ClaimCreated(AccountId,Vec<u8>),  // 用户AccountId，存证内容 Vec<u8>
		ClaimRevoked(AccountId,Vec<u8>),
	}
);

// Errors inform users that something went wrong.
decl_error! {
	pub enum Error for Module<T: Trait> {
		ProofAlreadyExist,    // 存在异常，即存证已经存在
		ClaimNotExist,
		NotClaimOwner,
	}
}

// Dispatchable functions allows users to interact with the pallet and invoke state changes.
// These functions materialize as "extrinsics", which are often compared to transactions.
// Dispatchable functions must be annotated with a weight and must return a DispatchResult.
decl_module! {
	pub struct Module<T: Trait> for enum Call where origin: T::Origin {
		// Errors must be initialized if they are used by the pallet.
		type Error = Error<T>;

		// Events must be initialized if they are used by the pallet.
		fn deposit_event() = default;

		// 创建存证，创建存证需要有两个关键参数：交易发送方origin，存证hash值claim，由于存证hash函数未知，也和decl_storage定义对应，这里使用变长Vec<u8>
        #[weight = 0]
		pub fn create_claim(origin,claim:Vec<u8>)->dispatch::DispatchResult{
			// 做必要检查，检查内容： 1，交易发送方是不是一个签名的用户 2，存证是否被别人创建过，创建过就抛出错误
			// 首先去创建签名交易，通过ensure_signed这样的system提供的版本方法来校验
			let sender = ensure_signed(origin)?;  // 存证拥有人是交易发送方，只有拥有人才可以调用存证，sender即当前交易发送方
  			// 如果存在存证，返回错误 ProofAlreadyExist
  			// ps:ensure!宏是确保表达式中的结果为true，这里取反操作
			ensure!(!Proofs::<T>::contains_key(&claim),Error::<T>::ProofAlreadyExist);  // 这里用到一个错误  ProofAlreadyExist，该错误需要在decl_error声明
			// 做insert操作，insert是key-value方式。这里的key-value是一个tuple
			// 这个tuple的第一个元素是AccountId；第二个是当前交易所处的区块，使用系统模块提供的block_number工具方法获取
			Proofs::<T>::insert(&claim,(sender.clone(),frame_system::Module::<T>::block_number()));  // 插入操作
			// 触发一个event来通知客户端，RawEvent由宏生成；   sender:存在拥有人；claim:存在hash值 通过event通知客户端
			Self::deposit_event(RawEvent::ClaimCreated(sender,claim));   // ClaimCreated事件，需要decl_event处理
			// 返回ok
			Ok(())
		}

		#[weight = 0]
		pub fn revoke_claim(origin,claim: Vec<u8>) -> dispatch::DispatchResult{
			let sender = ensure_signed(origin)?;  // 交易发送方式已签名的， 存证拥有人是交易发送方，只有拥有人才可以吊销存证

  			// 判断存储单元里面是存在这样一个存证；如果不存在，抛出错误，错误我们叫ClaimNotExist
			ensure!(Proofs::<T>::contains_key(&claim),Error::<T>::ClaimNotExist);

			// 获取这样的存证  owner: accountId   block_number
			let (owner,_block_number) = Proofs::<T>::get(&claim);  // 通过get api获取这样的一个存证

			ensure!(owner == sender,Error::<T>::NotClaimOwner);  // 确保交易发送方是我们的存证人，如果不是，返回Error，这个Error我们叫NotClaimOwner

			// 以上校验完成之后，我们就可以删除我们的存证
		    // 存储向上调用remove函数进行删除
		    Proofs::<T>::remove(&claim);

			// 触发一个事件，返回存证人和hash
		    Self::deposit_event(RawEvent::ClaimRevoked(sender,claim));

			// 返回
			Ok(())
		}
		
	}
}


2、pallets/poe/Cargo.toml****************************************************************************************************************************************
[package]
authors = ['Substrate DevHub <https://github.com/substrate-developer-hub>']
description = 'FRAME pallet proof of existence.'
edition = '2018'
homepage = 'https://substrate.dev'
license = 'Unlicense'
name = 'pallet-poe'
repository = 'https://github.com/substrate-developer-hub/substrate-node-template/'
version = '2.0.0'

[package.metadata.docs.rs]
targets = ['x86_64-unknown-linux-gnu']

# alias "parity-scale-code" to "codec"
[dependencies.codec]
default-features = false
features = ['derive']
package = 'parity-scale-codec'
version = '1.3.4'

[dependencies.sp-std]
default-features = false
git = 'https://github.com/paritytech/substrate.git'
tag = 'v2.0.0-rc5'
version = '2.0.0-rc5'

[dependencies]
frame-support = { default-features = false, version = '2.0.0' }
frame-system = { default-features = false, version = '2.0.0' }

[dev-dependencies]
sp-core = { default-features = false, version = '2.0.0' }
sp-io = { default-features = false, version = '2.0.0' }
sp-runtime = { default-features = false, version = '2.0.0' }

[features]
default = ['std']
std = [
    'codec/std',
    'frame-support/std',
    'frame-system/std',
    'sp-std/std',
]

3、pallets/runtime/src/lib.rs*********************************************************************************************************************************

#![cfg_attr(not(feature = "std"), no_std)]
// `construct_runtime!` does a lot of recursion and requires us to increase the limit to 256.
#![recursion_limit="256"]

// Make the WASM binary available.
#[cfg(feature = "std")]
include!(concat!(env!("OUT_DIR"), "/wasm_binary.rs"));

use sp_std::prelude::*;
use sp_core::{crypto::KeyTypeId, OpaqueMetadata};
use sp_runtime::{
	ApplyExtrinsicResult, generic, create_runtime_str, impl_opaque_keys, MultiSignature,
	transaction_validity::{TransactionValidity, TransactionSource},
};
use sp_runtime::traits::{
	BlakeTwo256, Block as BlockT, IdentityLookup, Verify, IdentifyAccount, NumberFor, Saturating,
};
use sp_api::impl_runtime_apis;
use sp_consensus_aura::sr25519::AuthorityId as AuraId;
use pallet_grandpa::{AuthorityId as GrandpaId, AuthorityList as GrandpaAuthorityList};
use pallet_grandpa::fg_primitives;
use sp_version::RuntimeVersion;
#[cfg(feature = "std")]
use sp_version::NativeVersion;

// A few exports that help ease life for downstream crates.
#[cfg(any(feature = "std", test))]
pub use sp_runtime::BuildStorage;
pub use pallet_timestamp::Call as TimestampCall;
pub use pallet_balances::Call as BalancesCall;
pub use sp_runtime::{Permill, Perbill};
pub use frame_support::{
	construct_runtime, parameter_types, StorageValue,
	traits::{KeyOwnerProofSystem, Randomness},
	weights::{
		Weight, IdentityFee,
		constants::{BlockExecutionWeight, ExtrinsicBaseWeight, RocksDbWeight, WEIGHT_PER_SECOND},
	},
};

/// Import the template pallet.
pub use pallet_template;

/// An index to a block.
pub type BlockNumber = u32;

/// Alias to 512-bit hash when used in the context of a transaction signature on the chain.
pub type Signature = MultiSignature;

/// Some way of identifying an account on the chain. We intentionally make it equivalent
/// to the public key of our transaction signing scheme.
pub type AccountId = <<Signature as Verify>::Signer as IdentifyAccount>::AccountId;

/// The type for looking up accounts. We don't expect more than 4 billion of them, but you
/// never know...
pub type AccountIndex = u32;

/// Balance of an account.
pub type Balance = u128;

/// Index of a transaction in the chain.
pub type Index = u32;

/// A hash of some data used by the chain.
pub type Hash = sp_core::H256;

/// Digest item type.
pub type DigestItem = generic::DigestItem<Hash>;

/// Opaque types. These are used by the CLI to instantiate machinery that don't need to know
/// the specifics of the runtime. They can then be made to be agnostic over specific formats
/// of data like extrinsics, allowing for them to continue syncing the network through upgrades
/// to even the core data structures.
pub mod opaque {
	use super::*;

	pub use sp_runtime::OpaqueExtrinsic as UncheckedExtrinsic;

	/// Opaque block header type.
	pub type Header = generic::Header<BlockNumber, BlakeTwo256>;
	/// Opaque block type.
	pub type Block = generic::Block<Header, UncheckedExtrinsic>;
	/// Opaque block identifier type.
	pub type BlockId = generic::BlockId<Block>;

	impl_opaque_keys! {
		pub struct SessionKeys {
			pub aura: Aura,
			pub grandpa: Grandpa,
		}
	}
}

pub const VERSION: RuntimeVersion = RuntimeVersion {
	spec_name: create_runtime_str!("node-template"),
	impl_name: create_runtime_str!("node-template"),
	authoring_version: 1,
	spec_version: 1,
	impl_version: 1,
	apis: RUNTIME_API_VERSIONS,
	transaction_version: 1,
};

pub const MILLISECS_PER_BLOCK: u64 = 6000;

pub const SLOT_DURATION: u64 = MILLISECS_PER_BLOCK;

// Time is measured by number of blocks.
pub const MINUTES: BlockNumber = 60_000 / (MILLISECS_PER_BLOCK as BlockNumber);
pub const HOURS: BlockNumber = MINUTES * 60;
pub const DAYS: BlockNumber = HOURS * 24;

/// The version information used to identify this runtime when compiled natively.
#[cfg(feature = "std")]
pub fn native_version() -> NativeVersion {
	NativeVersion {
		runtime_version: VERSION,
		can_author_with: Default::default(),
	}
}

parameter_types! {
	pub const BlockHashCount: BlockNumber = 2400;
	/// We allow for 2 seconds of compute with a 6 second average block time.
	pub const MaximumBlockWeight: Weight = 2 * WEIGHT_PER_SECOND;
	pub const AvailableBlockRatio: Perbill = Perbill::from_percent(75);
	/// Assume 10% of weight for average on_initialize calls.
	pub MaximumExtrinsicWeight: Weight = AvailableBlockRatio::get()
		.saturating_sub(Perbill::from_percent(10)) * MaximumBlockWeight::get();
	pub const MaximumBlockLength: u32 = 5 * 1024 * 1024;
	pub const Version: RuntimeVersion = VERSION;
}

// Configure FRAME pallets to include in runtime.

impl frame_system::Trait for Runtime {
	/// The basic call filter to use in dispatchable.
	type BaseCallFilter = ();
	/// The identifier used to distinguish between accounts.
	type AccountId = AccountId;
	/// The aggregated dispatch type that is available for extrinsics.
	type Call = Call;
	/// The lookup mechanism to get account ID from whatever is passed in dispatchers.
	type Lookup = IdentityLookup<AccountId>;
	/// The index type for storing how many extrinsics an account has signed.
	type Index = Index;
	/// The index type for blocks.
	type BlockNumber = BlockNumber;
	/// The type for hashing blocks and tries.
	type Hash = Hash;
	/// The hashing algorithm used.
	type Hashing = BlakeTwo256;
	/// The header type.
	type Header = generic::Header<BlockNumber, BlakeTwo256>;
	/// The ubiquitous event type.
	type Event = Event;
	/// The ubiquitous origin type.
	type Origin = Origin;
	/// Maximum number of block number to block hash mappings to keep (oldest pruned first).
	type BlockHashCount = BlockHashCount;
	/// Maximum weight of each block.
	type MaximumBlockWeight = MaximumBlockWeight;
	/// The weight of database operations that the runtime can invoke.
	type DbWeight = RocksDbWeight;
	/// The weight of the overhead invoked on the block import process, independent of the
	/// extrinsics included in that block.
	type BlockExecutionWeight = BlockExecutionWeight;
	/// The base weight of any extrinsic processed by the runtime, independent of the
	/// logic of that extrinsic. (Signature verification, nonce increment, fee, etc...)
	type ExtrinsicBaseWeight = ExtrinsicBaseWeight;
	/// The maximum weight that a single extrinsic of `Normal` dispatch class can have,
	/// idependent of the logic of that extrinsics. (Roughly max block weight - average on
	/// initialize cost).
	type MaximumExtrinsicWeight = MaximumExtrinsicWeight;
	/// Maximum size of all encoded transactions (in bytes) that are allowed in one block.
	type MaximumBlockLength = MaximumBlockLength;
	/// Portion of the block weight that is available to all normal transactions.
	type AvailableBlockRatio = AvailableBlockRatio;
	/// Version of the runtime.
	type Version = Version;
	/// Converts a module to the index of the module in `construct_runtime!`.
	///
	/// This type is being generated by `construct_runtime!`.
	type PalletInfo = PalletInfo;
	/// What to do if a new account is created.
	type OnNewAccount = ();
	/// What to do if an account is fully reaped from the system.
	type OnKilledAccount = ();
	/// The data to be stored in an account.
	type AccountData = pallet_balances::AccountData<Balance>;
	/// Weight information for the extrinsics of this pallet.
	type SystemWeightInfo = ();
}

impl pallet_aura::Trait for Runtime {
	type AuthorityId = AuraId;
}

impl pallet_grandpa::Trait for Runtime {
	type Event = Event;
	type Call = Call;

	type KeyOwnerProofSystem = ();

	type KeyOwnerProof =
		<Self::KeyOwnerProofSystem as KeyOwnerProofSystem<(KeyTypeId, GrandpaId)>>::Proof;

	type KeyOwnerIdentification = <Self::KeyOwnerProofSystem as KeyOwnerProofSystem<(
		KeyTypeId,
		GrandpaId,
	)>>::IdentificationTuple;

	type HandleEquivocation = ();

	type WeightInfo = ();
}

parameter_types! {
	pub const MinimumPeriod: u64 = SLOT_DURATION / 2;
}

impl pallet_timestamp::Trait for Runtime {
	/// A timestamp: milliseconds since the unix epoch.
	type Moment = u64;
	type OnTimestampSet = Aura;
	type MinimumPeriod = MinimumPeriod;
	type WeightInfo = ();
}

parameter_types! {
	pub const ExistentialDeposit: u128 = 500;
	pub const MaxLocks: u32 = 50;
}

impl pallet_balances::Trait for Runtime {
	type MaxLocks = MaxLocks;
	/// The type for recording an account's balance.
	type Balance = Balance;
	/// The ubiquitous event type.
	type Event = Event;
	type DustRemoval = ();
	type ExistentialDeposit = ExistentialDeposit;
	type AccountStore = System;
	type WeightInfo = ();
}

parameter_types! {
	pub const TransactionByteFee: Balance = 1;
}

impl pallet_transaction_payment::Trait for Runtime {
	type Currency = Balances;
	type OnTransactionPayment = ();
	type TransactionByteFee = TransactionByteFee;
	type WeightToFee = IdentityFee<Balance>;
	type FeeMultiplierUpdate = ();
}

impl pallet_sudo::Trait for Runtime {
	type Event = Event;
	type Call = Call;
}

/// Configure the template pallet in pallets/template.
impl pallet_template::Trait for Runtime {
	type Event = Event;
}

/// 对配置接口进行实现
impl poe::Trait for Runtime{
	type Event = Event;
}

// Create the runtime by composing the FRAME pallets that were previously configured.
construct_runtime!(
	pub enum Runtime where
		Block = Block,
		NodeBlock = opaque::Block,
		UncheckedExtrinsic = UncheckedExtrinsic
	{
		System: frame_system::{Module, Call, Config, Storage, Event<T>},
		RandomnessCollectiveFlip: pallet_randomness_collective_flip::{Module, Call, Storage},
		Timestamp: pallet_timestamp::{Module, Call, Storage, Inherent},
		Aura: pallet_aura::{Module, Config<T>, Inherent},
		Grandpa: pallet_grandpa::{Module, Call, Storage, Config, Event},
		Balances: pallet_balances::{Module, Call, Storage, Config<T>, Event<T>},
		TransactionPayment: pallet_transaction_payment::{Module, Storage},
		Sudo: pallet_sudo::{Module, Call, Config<T>, Storage, Event<T>},
		// Include the custom logic from the template pallet in the runtime.
		TemplateModule: pallet_template::{Module, Call, Storage, Event<T>},

		// 引入poe对应模块的操作信息  Module: 模块  Call：调用函数  Storage：存储项  Event<T>：事件  Error不需要额外引入
		PoeModule: poe::{Module, Call, Storage, Event<T>},
	}
);

/// The address format for describing accounts.
pub type Address = AccountId;
/// Block header type as expected by this runtime.
pub type Header = generic::Header<BlockNumber, BlakeTwo256>;
/// Block type as expected by this runtime.
pub type Block = generic::Block<Header, UncheckedExtrinsic>;
/// A Block signed with a Justification
pub type SignedBlock = generic::SignedBlock<Block>;
/// BlockId type as expected by this runtime.
pub type BlockId = generic::BlockId<Block>;
/// The SignedExtension to the basic transaction logic.
pub type SignedExtra = (
	frame_system::CheckSpecVersion<Runtime>,
	frame_system::CheckTxVersion<Runtime>,
	frame_system::CheckGenesis<Runtime>,
	frame_system::CheckEra<Runtime>,
	frame_system::CheckNonce<Runtime>,
	frame_system::CheckWeight<Runtime>,
	pallet_transaction_payment::ChargeTransactionPayment<Runtime>
);
/// Unchecked extrinsic type as expected by this runtime.
pub type UncheckedExtrinsic = generic::UncheckedExtrinsic<Address, Call, Signature, SignedExtra>;
/// Extrinsic type that has already been checked.
pub type CheckedExtrinsic = generic::CheckedExtrinsic<AccountId, Call, SignedExtra>;
/// Executive: handles dispatch to the various modules.
pub type Executive = frame_executive::Executive<
	Runtime,
	Block,
	frame_system::ChainContext<Runtime>,
	Runtime,
	AllModules,
>;

impl_runtime_apis! {
	impl sp_api::Core<Block> for Runtime {
		fn version() -> RuntimeVersion {
			VERSION
		}

		fn execute_block(block: Block) {
			Executive::execute_block(block)
		}

		fn initialize_block(header: &<Block as BlockT>::Header) {
			Executive::initialize_block(header)
		}
	}

	impl sp_api::Metadata<Block> for Runtime {
		fn metadata() -> OpaqueMetadata {
			Runtime::metadata().into()
		}
	}

	impl sp_block_builder::BlockBuilder<Block> for Runtime {
		fn apply_extrinsic(extrinsic: <Block as BlockT>::Extrinsic) -> ApplyExtrinsicResult {
			Executive::apply_extrinsic(extrinsic)
		}

		fn finalize_block() -> <Block as BlockT>::Header {
			Executive::finalize_block()
		}

		fn inherent_extrinsics(data: sp_inherents::InherentData) -> Vec<<Block as BlockT>::Extrinsic> {
			data.create_extrinsics()
		}

		fn check_inherents(
			block: Block,
			data: sp_inherents::InherentData,
		) -> sp_inherents::CheckInherentsResult {
			data.check_extrinsics(&block)
		}

		fn random_seed() -> <Block as BlockT>::Hash {
			RandomnessCollectiveFlip::random_seed()
		}
	}

	impl sp_transaction_pool::runtime_api::TaggedTransactionQueue<Block> for Runtime {
		fn validate_transaction(
			source: TransactionSource,
			tx: <Block as BlockT>::Extrinsic,
		) -> TransactionValidity {
			Executive::validate_transaction(source, tx)
		}
	}

	impl sp_offchain::OffchainWorkerApi<Block> for Runtime {
		fn offchain_worker(header: &<Block as BlockT>::Header) {
			Executive::offchain_worker(header)
		}
	}

	impl sp_consensus_aura::AuraApi<Block, AuraId> for Runtime {
		fn slot_duration() -> u64 {
			Aura::slot_duration()
		}

		fn authorities() -> Vec<AuraId> {
			Aura::authorities()
		}
	}

	impl sp_session::SessionKeys<Block> for Runtime {
		fn generate_session_keys(seed: Option<Vec<u8>>) -> Vec<u8> {
			opaque::SessionKeys::generate(seed)
		}

		fn decode_session_keys(
			encoded: Vec<u8>,
		) -> Option<Vec<(Vec<u8>, KeyTypeId)>> {
			opaque::SessionKeys::decode_into_raw_public_keys(&encoded)
		}
	}

	impl fg_primitives::GrandpaApi<Block> for Runtime {
		fn grandpa_authorities() -> GrandpaAuthorityList {
			Grandpa::grandpa_authorities()
		}

		fn submit_report_equivocation_unsigned_extrinsic(
			_equivocation_proof: fg_primitives::EquivocationProof<
				<Block as BlockT>::Hash,
				NumberFor<Block>,
			>,
			_key_owner_proof: fg_primitives::OpaqueKeyOwnershipProof,
		) -> Option<()> {
			None
		}

		fn generate_key_ownership_proof(
			_set_id: fg_primitives::SetId,
			_authority_id: GrandpaId,
		) -> Option<fg_primitives::OpaqueKeyOwnershipProof> {
			// NOTE: this is the only implementation possible since we've
			// defined our key owner proof type as a bottom type (i.e. a type
			// with no values).
			None
		}
	}

	impl frame_system_rpc_runtime_api::AccountNonceApi<Block, AccountId, Index> for Runtime {
		fn account_nonce(account: AccountId) -> Index {
			System::account_nonce(account)
		}
	}

	impl pallet_transaction_payment_rpc_runtime_api::TransactionPaymentApi<Block, Balance> for Runtime {
		fn query_info(
			uxt: <Block as BlockT>::Extrinsic,
			len: u32,
		) -> pallet_transaction_payment_rpc_runtime_api::RuntimeDispatchInfo<Balance> {
			TransactionPayment::query_info(uxt, len)
		}
	}

	#[cfg(feature = "runtime-benchmarks")]
	impl frame_benchmarking::Benchmark<Block> for Runtime {
		fn dispatch_benchmark(
			config: frame_benchmarking::BenchmarkConfig
		) -> Result<Vec<frame_benchmarking::BenchmarkBatch>, sp_runtime::RuntimeString> {
			use frame_benchmarking::{Benchmarking, BenchmarkBatch, add_benchmark, TrackedStorageKey};

			use frame_system_benchmarking::Module as SystemBench;
			impl frame_system_benchmarking::Trait for Runtime {}

			let whitelist: Vec<TrackedStorageKey> = vec![
				// Block Number
				hex_literal::hex!("26aa394eea5630e07c48ae0c9558cef702a5c1b19ab7a04f536c519aca4983ac").to_vec().into(),
				// Total Issuance
				hex_literal::hex!("c2261276cc9d1f8598ea4b6a74b15c2f57c875e4cff74148e4628f264b974c80").to_vec().into(),
				// Execution Phase
				hex_literal::hex!("26aa394eea5630e07c48ae0c9558cef7ff553b5a9862a516939d82b3d3d8661a").to_vec().into(),
				// Event Count
				hex_literal::hex!("26aa394eea5630e07c48ae0c9558cef70a98fdbe9ce6c55837576c60c7af3850").to_vec().into(),
				// System Events
				hex_literal::hex!("26aa394eea5630e07c48ae0c9558cef780d41e5e16056765bc8461851072c9d7").to_vec().into(),
			];

			let mut batches = Vec::<BenchmarkBatch>::new();
			let params = (&config, &whitelist);

			add_benchmark!(params, batches, frame_system, SystemBench::<Runtime>);
			add_benchmark!(params, batches, pallet_balances, Balances);
			add_benchmark!(params, batches, pallet_timestamp, Timestamp);

			if batches.is_empty() { return Err("Benchmark not found for this pallet.".into()) }
			Ok(batches)
		}
	}
}


4、pallets/runtime/Cargo.toml **************************************************************************************************************************************************

[package]
authors = ['Substrate DevHub <https://github.com/substrate-developer-hub>']
edition = '2018'
homepage = 'https://substrate.dev'
license = 'Unlicense'
name = 'node-template-runtime'
repository = 'https://github.com/substrate-developer-hub/substrate-node-template/'
version = '2.0.0'

[package.metadata.docs.rs]
targets = ['x86_64-unknown-linux-gnu']

[build-dependencies]
wasm-builder-runner = { package = 'substrate-wasm-builder-runner', version = '2.0.0' }

# alias "parity-scale-code" to "codec"
[dependencies.codec]
default-features = false
features = ['derive']
package = 'parity-scale-codec'
version = '1.3.4'

# 引入poe依赖
[dependencies.poe]
default-features = false
package = 'pallet-poe'
path = '../pallets/poe'
version = '2.0.0-rc2'

[dependencies]
hex-literal = { optional = true, version = '0.3.1' }
serde = { features = ['derive'], optional = true, version = '1.0.101' }

# local dependencies
pallet-template = { path = '../pallets/template', default-features = false, version = '2.0.0' }

# Substrate dependencies
frame-benchmarking = { default-features = false, optional = true, version = '2.0.0' }
frame-executive = { default-features = false, version = '2.0.0' }
frame-support = { default-features = false, version = '2.0.0' }
frame-system = { default-features = false, version = '2.0.0' }
frame-system-benchmarking = { default-features = false, optional = true, version = '2.0.0' }
frame-system-rpc-runtime-api = { default-features = false, version = '2.0.0' }
pallet-aura = { default-features = false, version = '2.0.0' }
pallet-balances = { default-features = false, version = '2.0.0' }
pallet-grandpa = { default-features = false, version = '2.0.0' }
pallet-randomness-collective-flip = { default-features = false, version = '2.0.0' }
pallet-sudo = { default-features = false, version = '2.0.0' }
pallet-timestamp = { default-features = false, version = '2.0.0' }
pallet-transaction-payment = { default-features = false, version = '2.0.0' }
pallet-transaction-payment-rpc-runtime-api = { default-features = false, version = '2.0.0' }
sp-api = { default-features = false, version = '2.0.0' }
sp-block-builder = { default-features = false, version = '2.0.0' }
sp-consensus-aura = { default-features = false, version = '0.8.0' }
sp-core = { default-features = false, version = '2.0.0' }
sp-inherents = { default-features = false, version = '2.0.0' }
sp-offchain = { default-features = false, version = '2.0.0' }
sp-runtime = { default-features = false, version = '2.0.0' }
sp-session = { default-features = false, version = '2.0.0' }
sp-std = { default-features = false, version = '2.0.0' }
sp-transaction-pool = { default-features = false, version = '2.0.0' }
sp-version = { default-features = false, version = '2.0.0' }

[features]
default = ['std']
runtime-benchmarks = [
    'hex-literal',
    'frame-benchmarking',
    'frame-support/runtime-benchmarks',
    'frame-system-benchmarking',
    'frame-system/runtime-benchmarks',
    'pallet-balances/runtime-benchmarks',
    'pallet-timestamp/runtime-benchmarks',
    'sp-runtime/runtime-benchmarks',
]
std = [
    'codec/std',
    'serde',
    'frame-executive/std',
    'frame-support/std',
    'frame-system/std',
    'frame-system-rpc-runtime-api/std',
    'pallet-aura/std',
    'pallet-balances/std',
    'pallet-grandpa/std',
    'pallet-randomness-collective-flip/std',
    'pallet-sudo/std',
    'pallet-template/std',
    'pallet-timestamp/std',
    'pallet-transaction-payment/std',
    'pallet-transaction-payment-rpc-runtime-api/std',
    'sp-api/std',
    'sp-block-builder/std',
    'sp-consensus-aura/std',
    'sp-core/std',
    'sp-inherents/std',
    'sp-offchain/std',
    'sp-runtime/std',
    'sp-session/std',
    'sp-std/std',
    'sp-transaction-pool/std',
    'sp-version/std',
    'poe/std',  # 增加标签
]

