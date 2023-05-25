# Module System

This directory contains an opinionated framework for building rollups with the Sovereign SDK. It aims to provide a
"batteries included" development experience. Using the module system still allows you to customize key components of your rollup
like its hash function and signature scheme, but it also forces you to rely on some reasonable default values for things like
serialization schemes (Borsh) address formats (bech32), etc.

By developing with the module system, you get access to a suite of pre-built modules supporting common functions like generating accounts,
minting and transferring tokens, and incentivizing sequencers. You also get access to powerful tools for generating RPC implementations,
and a powerful templating system for implementing complex state transitions.

## Modules: The Basic Building Block

The basic building block of the module system is a `module`. Modules are structs in Rust, and are _required_ to implement the `Module` trait.
You can find a complete tutorial showing how to implement a custom module [here](../examples/demo-nft-module/README.md).
Modules typically live in their own crates (you can find a template [here](./module-implementations/module-template/)) so that they're easily
re-usable. A typical struct definition for a module looks something like this:

```rust
#[derive(ModuleInfo)]
pub struct Bank<C: sov_modules_api::Context> {
    /// The address of the bank module.
    #[address]
    pub(crate) address: C::Address,

    /// A mapping of addresses to tokens in the bank.
    #[state]
    pub(crate) tokens: sov_state::StateMap<C::Address, Token<C>>,
}
```

At first glance, this definition might seem a little bit intimidating because of the generic `C`. Don't worry, we'll explain that
generic in detail later. For now, just notice that a module is a struct with an address and some `#[state]` fields specifying
what kind of data this module has access to. Under the hood, the `ModuleInfo` derive macro will do some magic to ensure that
any `#[state]` fields get mapped onto unique storage keys so that only this particular module can read or write its state values.

At this stage, it's also very important to note that the state values are external to the module. This struct definition defines the
_shape_ of the values that will be stored, but the values themselves don't live inside the module struct. In other words, a module doesn't
secretly have a reference to some underlying database. Instead a module defines the _logic_ used to access state values,
and the values themselves live in a special struct called a `WorkingSet`.

This has several consequences. First, it means that modules are always cheap to clone. Second it means that calling `my_module.clone()`
always yields the same result as calling `MyModule::new()`. Finally, it means that every method of the module which reads or
modifies state needs to take a `WorkingSet` as an argument.

### Public Functions: The Module-to-Module Interface

The first interface that modules expose is defined by the public methods from the rollup's `impl`. These methods are
accessible to other modules, but cannot be directly invoked by other users. A good example of this is the `bank.transfer_from` method:

```rust
impl<C: Context> Bank<C> {
    pub fn transfer_from(&self, from: &C::Address, to: &C::Address, coins: Coins, working_set: &mut WorkingSet<C::Storage>) {
        // Implementation elided...
    }
}
```

This function transfers coins from one address to another _without a signature check_. If it was exposed to users, it would allow
for the theft of funds. But it's very useful for modules to be able to initiate funds transfers without access to the private keys
of their own the owning accounts. (Of course, modules should be careful to get the user's consent before transferring funds. By
using the transfer_from interface, a module is declaring that it has gotten such consent.)

This leads us to a very important point about the module system. All modules are _trusted_. Unlike smart contracts on Ethereum, modules
cannot be dynamically deployed by users - they're fixed up-front by the rollup developer. That doesn't mean that the Sovereign SDK doesn't
support smart contracts - just that they live one layer higher up the stack. If you want to deploy smart contracts on your rollup, you'll need
to incorporate a _module_ which implements a secure virtual machine that users can invoke to store and run smart contracts.

### The `Call` Function: The Module-to-User Interface

The second interface exposed by modules is the `call` function from the `Module` trait. The `call` function defines the
interface which is _accessible to users via on-chain transactions_, and it typically takes an enum as its first argument. This argument
tells the `call` function which inner method of the module to invoke. So a typical implementation of `call` looks something like this:

```rust
impl<C: sov_modules_api::Context> sov_modules_api::Module for Bank<C> {
	// Several definitions elided here ...
    fn call(&self, msg: Self::CallMessage, context: &Self::Context, working_set: &mut WorkingSet<C::Storage>) {
        match msg {
            CallMessage::CreateToken {
                token_name,
                minter_address,
            } => Ok(self.create_token(token_name, minter_address, context, working_set)?),
            CallMessage::Transfer { to, coins } => { Ok(self.transfer(to, coins, context, working_set)?) },
            CallMessage::Burn { coins } => Ok(self.burn(coins, context, working_set)?),
        }
    }
}
```

### The `RPC` Macro: The Node-to-User Interface

The third interface that modules expose is an rpc implementation. To generate an RPC implementation, simply annotate your `impl` block
with the `#[rpc_gen]` macro from `sov_modules_macros`.

```rust
#[rpc_gen(client, server, namespace = "sov-bank")]
impl<C: sov_modules_api::Context> Bank<C> {
    #[rpc_method(name = "balanceOf")]
    pub(crate) fn balance_of(
        &self,
        user_address: C::Address,
        token_address: C::Address,
        working_set: &mut WorkingSet<C::Storage>,
    ) ->  
        BalanceResponse {
            amount: self.get_balance_of(user_address, token_address, working_set),
        }
    }
}
```

This will generate a public trait in the bank crate called `BankRpcImpl`, which understands how to serve requests with the following form:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "bank_balanceOf",
  "params": { "user_address": "SOME_ADDRESS", "token_address": "SOME_ADDRESS" }
}
```

For an example of how to instantiate the generated trait as a server bound to a specific port, see the [demo-rollup](../examples/demo-rollup/) package.

Note that only one impl block per module may be annotated with `rpc_gen`, but that the block may contain as many `rpc_method` annotations as you want.

## Context and Spec: How to Make Your Module System Portable

In addition to `Module`, there are two traits that are ubiquitous in the modules system - `Context` and `Spec`. To understand these
two traits it's useful to remember that the high-level workflow of a Sovereign SDK rollup consists of two stages.
First, transactions are executed in native code to generate a "witness". Then, the witness is fed to the zk-circuit,
which re-executes the transactions in a (more expensive) zk environment to create a proof. So, pseudocode for the rollup
workflow looks roughly like this:

```rust
// First, execute transactions natively to generate a witness for the zkvm
let native_rollup_instance = my_state_transition::<DefaultContext>::new(config);
let witness = Default::default()
native_rollup_instance.begin_slot(witness);
for batch in batches.cloned() {
	native_rollup_instance.apply_batch(batch);
}
let (_new_state_root, populated_witness) = native_rollup_instance.end_batch();

// Then, re-execute the state transitions in the zkvm using the witness
let proof = MyZkvm::prove(|| {
	let zk_rollup_instance = my_state_transition::<ZkDefaultContext>::new(config);
	zk_rollup_instance.begin_slot(populated_witness);
	for batch in batches {
		zk_rollup_instance.apply(batch);
	}
	let (new_state_root, _) = zk_rollup_instance.end_batch();
	MyZkvm::commit(new_state_root)
})
```

This distinction between native _execution_ and zero-knowledge _re-execution_ is deeply baked into the module system. We take the
philosophy that your business logic should be identical whichever "mode" you're using, so we abstract the differences between
the zk and native modes behind a few traits.

### Using traits to Customize Your Behavior for Different Modes

The most important trait we use to enable this abstraction is the `Spec` trait. A (simplified) `Spec` is defined like this:

```rust
pub trait Spec {
    type Storage;
    type PublicKey;
    type Hasher;
    type Signature;
}
```

As you can see, a `Spec` for a rollup specifies the concrete types that will be used for many kinds of cryptographic operations.
That way, you can define your business logic in terms of _abstract_ cryptography, and then instantiate it with cryptography which
is efficient in your particular choice of zkvm.

In addition to the `Spec` trait, the module system provides a simple `Context` trait which is defined like this:

```rust
pub trait Context: Spec + Clone + Debug + PartialEq {
    /// Sender of the transaction.
    fn sender(&self) -> &Self::Address;
    /// Constructor for the Context.
    fn new(sender: Self::Address) -> Self;
}
```

Modules are expected to be generic over the `Context` type. This trait gives them a convenient handle to access all of the cryptographic operations
defined by a `Spec`, while also making it easy for the module system to pass in authenticated transaction-specific information which
would not otherwise be available to a module. Currently, a `Context` is only required to contain the `sender` (signer) of the transaction,
but this trait might be extended in future.

Putting it all together, recall that the Bank struct is defined like this.

```rust
pub struct Bank<C: sov_modules_api::Context> {
    /// The address of the bank module.
    pub(crate) address: C::Address,

    /// A mapping of addresses to tokens in the bank.
    pub(crate) tokens: sov_state::StateMap<C::Address, Token<C>>,
}
```

Notice that the generic type `C` is required to implement the `sov_modules_api::Context` trait. Thanks to that generic, the Bank struct can
access the `Address` field from `Spec` - meaning that your bank logic doesn't change if you swap out your underlying address schema.

Similarly, since each of the banks helper functions is automatically generic over a context, it's easy to define logic which
can abstract away the distinctions between `zk` and `native` execution. For example, when a rollup is running in native mode
its `Storage` type will almost certainly be [`ProverStorage`](./sov-state/src/prover_storage.rs), which holds its data in a
merkle tree backed by RocksDB. But if you're running in zk mode the `Storage` type will instead be `ZkStorage`, which reads
its data from a set of "hints" provided by the prover. Because all of the rollups modules are generic, none of them need to worry
about this distinction.

For more information on `Context` and `Spec`, and to see some example implementations, check out the [`sov_modules_api`](./sov-modules-api/) docs.


## Enabling RPC via SDK Macros

There are 5 steps that need to be completed to enable RPC on the full node
1. Annotate the modules that need to expose their data with `rpc_gen` and `rpc_method`
2. Annotate the state transition runner with the specific modules to expose with `expose_rpc`
3. Implement the `RpcRunner` trait. provide an implementation for the `get_storage` function
4. Inside the full node implementation - Import and call `get_rpc_methods` to get combined rpc methods for the modules annotated and exposed in 1 and 2
5. Inside the full node implementation - Use the modules returned from the above function and bind them to an RPC server

### Modules
* We need to annotate the `impl` block for our module. In this case its `Bank`
```rust
impl<C: Context> Bank<C> {
    pub(crate) fn balance_of(
        &self,
        user_address: C::Address,
        token_address: C::Address,
        working_set: &mut WorkingSet<C::Storage>,
    ) -> BalanceResponse {
    ...
    }
    
    pub(crate) fn supply_of(
        &self,
        token_address: C::Address,
        working_set: &mut WorkingSet<C::Storage>,
    ) -> TotalSupplyResponse {
     ...
    }
}
```
* annotate with `rpc_gen` and `rpc_method`
```rust
use sov_modules_macros::rpc_gen;

#[rpc_gen(client, server, namespace = "sov-bank")]
impl<C: Context> Bank<C> {
    #[rpc_method(name = "balanceOf")]
    pub(crate) fn balance_of(
        &self,
        user_address: C::Address,
        token_address: C::Address,
        working_set: &mut WorkingSet<C::Storage>,
    ) ->  BalanceResponse {
    ...
    }
    
    #[rpc_method(name = "supplyOf")]
    pub(crate) fn supply_of(
        &self,
        token_address: C::Address,
        working_set: &mut WorkingSet<C::Storage>,
    ) -> TotalSupplyResponse {
     ...
    }
}
```
* `rpc_gen` and `rpc_method` create <module_name>RpcImpl and <module_name>RpcServer traits.
* The ___RpcImpl and ___RpcServer traits do not need to be implemented - this is done automatically by the SDK, but they need to be imported to the file where the `expose_rpc` macro is called
* Once all the modules that need be part of the RPC are annotated, we annotate our Runner struct that impls `StateTransitionRunner` with an `expose_rpc` attribute macro.
```rust
use sov_bank::query::{BankRpcImpl, BankRpcServer};

#[expose_rpc((Bank<DefaultContext>,))]
impl<Vm: Zkvm> StateTransitionRunner<ProverConfig, Vm> for DemoAppRunner<DefaultContext, Vm> {
...
}
```
* `expose_rpc` takes a tuple as arguments. each element of the tuple is a module with a concrete Context.
* next, we implement the `RpcRunner` trait. we do this in the `demo_stf/app.rs` file
```rust
use sov_modules_api::RpcRunner;
impl<Vm: Zkvm> RpcRunner for DemoAppRunner<DefaultContext, Vm> {
    type Context = DefaultContext;
    fn get_storage(&self) -> <Self::Context as Spec>::Storage {
        self.inner().current_storage.clone()
    }
}
```
* `RpcRunner` primarily need to provide a storage which is used by the RPC server. It's a helper trait
* To start the jsonrpsee server, we need the rpc modules, which are provided by the macro generated method `get_rpc_methods`
```rust
use demo_stf::app::get_rpc_methods;

    let mut demo_runner = NativeAppRunner...;

    let storj = demo_runner.get_storage();
    let methods = get_rpc_methods(storj);
```
* This is the register + network interface binding step, and starting the actual RPC server
```rust
async fn start_rpc_server(methods: RpcModule<()>, address: SocketAddr) {
    let server = jsonrpsee::server::ServerBuilder::default()
        .build([address].as_ref())
        .await
        .unwrap();
    let _server_handle = server.start(methods).unwrap();
    futures::future::pending::<()>().await;
}  

    let _handle = tokio::spawn(async move {
        start_rpc_server(methods, address).await;
    }); 

```
* we're using `futures::future::pending::<()>().await` to block the spawned RPC server, but this can be implemented in multiple ways
* Another note is that we're configuring address in the `rollup_config.toml`
```toml
[rpc_config]
# the host and port to bind the rpc server for
bind_host = "127.0.0.1"
bind_port = 12345
```
* The above can be parsed using
```rust
    let rollup_config: RollupConfig = from_toml_path("rollup_config.toml")?;
    let rpc_config = rollup_config.rpc_config;
    let address = SocketAddr::new(rpc_config.bind_host.parse()?, rpc_config.bind_port);
```
* But as mentioned, the infra / networking aspect is separated from the macro that generates the boilerplate to expose the RPC in a way that it can be plugged into an RPC server