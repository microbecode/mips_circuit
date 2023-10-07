# Mips Circuit


This is a proof of concept (PoC) for zkmips implemented using the Cannon simulator, Zokrates DSL, and Groth16. It supports the full execution of Minigeth, outputs the entire instruction sequence, generates proofs for each instruction, and submits them to an on-chain contract for verification.

Unlike other zkVM projects, we didn't rush to define the proof system we needed initially. Instead, through this PoC, we aimed to understand the scale and challenges of the proofs required for using zkmips as a zk virtual machine for general computation. Now that this PoC is ready, we plan to release a testnet version based on an entirely new proof system in six months.
## Prerequisite

- Hardware:  MEM >= 8G

- Install [Rust](https://www.rust-lang.org/tools/install)

- Install [Go(>=1.21)](https://go.dev/doc/install)

- Install Make

- Install [Zokrates](https://zokrates.github.io/gettingstarted.html) , then set $ZOKRATES_STDLIB and $PATH (will be used in following compile MIPS VM circuit)

  ```
  export ZOKRATES_STDLIB= <path-to>/ZoKrates/zokrates_stdlib/stdlib
  export PATH=<path-to>/ZoKrates/target/release:$PATH
  ```

- Postgres DB

  - Install [postgres](https://www.postgresql.org/download/)  **NOTE: cannot use default empty password**
  - Install [pgadmin(optional)](https://www.pgadmin.org/download/) :Using the pgadmin GUI,it could manage the database visual graphically and easily.
  - Create Tables:

  ```
  DROP TABLE IF EXISTS f_traces;
  CREATE TABLE f_traces
  (
      f_id           bigserial PRIMARY KEY,
      f_trace        jsonb                    NOT NULL,
      f_created_at   TIMESTAMP with time zone NOT NULL DEFAULT now()
  );
  
  DROP TABLE IF EXISTS t_block_witness_cloud;
  CREATE TABLE t_block_witness_cloud
  (
      f_id             bigserial PRIMARY KEY,
      f_block          BIGINT NOT NULL,
      f_version        BIGINT NOT NULL,
      f_object_key     text   NOT NULL,
      f_object_witness text   NOT NULL
  );
  
  DROP TABLE IF EXISTS t_prover_job_queue_cloud;
  CREATE TABLE t_prover_job_queue_cloud
  (
      f_id           bigserial PRIMARY KEY,
      f_job_status   INTEGER                  NOT NULL,
      f_job_priority INTEGER                  NOT NULL,
      f_job_type     TEXT                     NOT NULL,
      f_created_at   timestamp with time zone NOT NULL DEFAULT now(),
      f_version      BIGINT                   NOT NULL,
      f_updated_by   TEXT                     NOT NULL,
      f_updated_at   timestamp with time zone NOT NULL,
      f_first_block  BIGINT                   NOT NULL,
      f_last_block   BIGINT                   NOT NULL,
      f_object_key   TEXT                     NOT NULL,
      f_object_job   TEXT                     NOT NULL
  );
  
  DROP TABLE IF EXISTS t_proofs;
  CREATE TABLE t_proofs
  (
      f_id           bigserial PRIMARY KEY,
      f_block_number BIGINT                   NOT NULL,
      f_proof        jsonb                    NOT NULL,
      f_created_at   TIMESTAMP with time zone NOT NULL DEFAULT now()
  );
  
  DROP TABLE IF EXISTS t_witness_block_number;
  CREATE TABLE t_witness_block_number
  (
      f_id           bigserial PRIMARY KEY,
      f_block     BIGINT                   NOT NULL
  );
  
  DROP TABLE IF EXISTS t_proof_block_number;
  CREATE TABLE t_proof_block_number
  (
      f_id           bigserial PRIMARY KEY,
      f_block     BIGINT                   NOT NULL
  );
  
  DROP TABLE IF EXISTS t_verified_proof_block_number;
  CREATE TABLE t_verified_proof_block_number
  (
      f_id           bigserial PRIMARY KEY,
      f_block       BIGINT                    NOT NULL
  );
  ```

  The id of first execution trace to be verified or proved  is 1 as default, you can specify your own <first_execution_trace_id> by following commands:

  ```
  INSERT INTO t_witness_block_number(f_block) VALUES(${<first_execution_trace_id>});
  
  INSERT INTO t_proof_block_number(f_block) VALUES(${$(<first_execution_trace_id>});
  ```

- Program Execution Records

  Build cannon-mips and generating the program execution record by [cannon-mips](https://github.com/zkMIPS/cannon-mips/tree/main#readme)

- Compile MIPS VM circuit using Zokrates 

  ```
  $ pushd core/lib/circuit
  $ zokrates compile -i mips_vm_poseidon.zok  # may need several minutes
  $ wget http://ec2-46-51-227-198.ap-northeast-1.compute.amazonaws.com/proving.key -O proving.key
  $ popd
  ```

## Deploy Verifier Contract

We have deployed a goerli verify contract at: 0xacd47ec395668320770e7183b9ee817f4ff8774e. Developers can use this to verify the proof. 

The transactions can be seen in https://goerli.etherscan.io/address/0xacd47ec395668320770e7183b9ee817f4ff8774e.


## Witness Generator

1. Setting the environment variables:

   ```
   $ source ./setenv.bash
   ```

   Or add following code to .bashrc file

   ```
   export DATABASE_URL=postgresql://<user>:<password>@<ip>:<port>/<db> 
   export DATABASE_POOL_SIZE=10
   export API_PROVER_PORT=8088
   export API_PROVER_URL=http://127.0.0.1:8088
   export API_PROVER_SECRET_AUTH=sample
   export PROVER_PROVER_HEARTBEAT_INTERVAL=1000
   export PROVER_PROVER_CYCLE_WAIT=500
   export PROVER_PROVER_REQUEST_TIMEOUT=10
   export PROVER_PROVER_DIE_AFTER_PROOF=false
   export PROVER_CORE_GONE_TIMEOUT=60000
   export PROVER_CORE_IDLE_PROVERS=1
   export PROVER_WITNESS_GENERATOR_PREPARE_DATA_INTERVAL=500
   export PROVER_WITNESS_GENERATOR_WITNESS_GENERATORS=2
   export CIRCUIT_FILE_PATH=${PWD}/core/lib/circuit/out # generated by zokrates compile -i mips_vm_poseidon.zok
   export CIRCUIT_ABI_FILE_PATH=${PWD}/core/lib/circuit/abi.json # generated by zokrates compile -i mips_vm_poseidon.zok
   export RUST_LOG=warn
   export VERIFIER_CHAIN_URL=https://eth-goerli.g.alchemy.com/v2/aLS5R8CYWcswzRyfKtGDDQD_noFqseN5 # chain url where the verifier contract deployed, Note: please use your own secret key here
   export VERIFIER_CONTRACT_ADDRESS=0xacd47ec395668320770e7183b9ee817f4ff8774e # verifier contract address
   export VERIFIER_ACCOUNT=b75dc70f894ef8bbd8cbb6d9f70c146b87f53cdb959f0ab6ac272a8b33e767f2 # your goerli account private key
   export VERIFIER_ABI_PATH=${PWD}/contract/verifier/g16/verifier
   export CHAIN_ETH_NETWORK=rinkeby
   export CIRCUIT_PROVING_KEY_PATH=${PWD}/core/lib/circuit/proving.key # generated by: zokrates compile -i mips_vm_poseidon.zok
   ```

   **NOTE: you should use your own Postgres database url to replace the relative fields in DATABASE_URL variable in both cases, and you should use your own VERIFIER_ACCOUNT, and may need to use your own node in VERIFIER_CHAIN_URL, such as register one from https://www.alchemy.com/ .**

2. Compile the witness generator

   ```
   $ pushd core/bin/server
   $ cargo build --release # may need several minutes
   $ popd
   ```

3. Running the witness generator

   ```
   $ nohup ./target/release/server > server.output 2>&1 &
   ```

## Prover

1. Setting the environment variables

   ```
   $ source ./setenv.bash
   ```

2. Compile the prover 

   ```
   $ pushd core/bin/prover
   $ cargo build --release  # may need several minutes
   $ popd
   ```

3. Running the prover

   ```
   $ nohup ./target/release/prover > prover.output 2>&1 &
   ```

   