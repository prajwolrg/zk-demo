## Installation
Instructions [here](https://docs.circom.io/getting-started/installation/)
## Guide
### 1. Create the circuit
```sh
cat <<EOT > circuit.circom
pragma circom 2.0.0;

template Multiplier2() {
    signal input a;
    signal input b;
    signal output c;

    c <== a * b;
}

component main = Multiplier2();
EOT
```

### 2. Compile the circuit
```sh
circom circuit.circom --r1cs --wasm --sym
```

The `circom` command takes one input (the circuit to compile, in our case `circuit.circom`) and three options:

- `r1cs`: generates `circuit.r1cs` (the r1cs constraint system of the circuit in binary format).

- `wasm`: generates `circuit.wasm` (the wasm code to generate the witness – more on that later).

- `sym`: generates `circuit.sym` (a symbols file required for debugging and printing the constraint system in an annotated mode).


#### View information about the circuit
```sh
snarkjs r1cs info circuit.r1cs
```

The `info` command is used to print circuit stats.

You should see the following output:

```
[INFO]  snarkJS: Curve: bn-128
[INFO]  snarkJS: # of Wires: 4
[INFO]  snarkJS: # of Constraints: 1
[INFO]  snarkJS: # of Private Inputs: 2
[INFO]  snarkJS: # of Public Inputs: 0
[INFO]  snarkJS: # of Labels: 4
[INFO]  snarkJS: # of Outputs: 1
```

This information fits with our mental map of the circuit we created: we had two private inputs `a` and `b`, one output `c`, and a single constraint of the form `a * b = c.`

### 3. Perform the trusted setup
This includes various steps including:
1. [Starting a new powers of tau ceremony](https://github.com/iden3/snarkjs#1-start-a-new-powers-of-tau-ceremony)
2. [Contributing to the ceremony](https://github.com/iden3/snarkjs#2-contribute-to-the-ceremony)
3. [Contributing to the with third party software](https://github.com/iden3/snarkjs#4-provide-a-third-contribution-using-third-party-software)
4. [Verifying the protocol so far](https://github.com/iden3/snarkjs#5-verify-the-protocol-so-far)
5. [Applying a random beacon](https://github.com/iden3/snarkjs#6-apply-a-random-beacon)
6. [Preparing for the phase 2](https://github.com/iden3/snarkjs#7-prepare-phase-2)

We will be using the powers of tau for bn128 curve with 54 contributions and a beacon used in the Hermez Powers of Tau ceremony. 
Since we only have 1 constraint, we can use the powers of tau that supports max 256 constraints.

```sh
wget https://hermez.s3-eu-west-1.amazonaws.com/powersOfTau28_hez_final_08.ptau
```

### 4. Generate the zkey

Currently, snarkjs supports 2 proving systems: groth16 and PLONK. 

Groth16 requires a trusted ceremony for each circuit. PLONK does not require it, it's enough with the powers of tau ceremony which is universal.
So, we'll be using PLONK for this demo
```sh
snarkjs plonk setup circuit.r1cs powersOfTau28_hez_final_08.ptau circuit_final.zkey
```
The zkey is a zero-knowledge key that includes both the proving and verification keys as well as phase 2 contributions.

Importantly, one can verify whether a zkey belongs to a specific circuit or not.

### 5. Extract verifying key from the zkey and export it as a solidity file
```sh
snarkjs zkey export solidityverifier circuit_final.zkey verifier.sol
```

We export the verifying key as a Solidity smart-contract so that we can publish it on-chain -- using [remix](https://remix.ethereum.org/) for example. For the details on how to do this, refer to section 4 of [this tutorial](https://blog.iden3.io/first-zk-proof.html).


### 6. Calculate the witness

First, we create a file with the inputs for our circuit:

```sh
cat <<EOT > input.json
{"a": 2, "b": 10}
EOT
```

Now, we use the Javascript/WASM program created by `circom` in the directory *circuit_js* to create the witness (values of all the wires) for our inputs:

```sh
circuit_js$ node generate_witness.js circuit.wasm ../input.json ../witness.wtns
```

#### Display the witness
```sh
snarkjs wtns export json witness.wtns witness.json
```

### 7. Create the proof

```sh
snarkjs plonk prove circuit_final.zkey witness.wtns proof.json public.json
```

#### Export soliditycalldata
```sh
snarkjs zkey export soliditycalldata public.json proof.json
```

We use `soliditycalldata` to simulate a verification call, and cut and paste the result directly in the verifyProof field in the deployed smart contract in the remix environment.

### 24. Verify the proof on-chain