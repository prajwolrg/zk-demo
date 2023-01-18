## Guide

### 0. Download the hermez powers of tau
```sh
wget https://hermez.s3-eu-west-1.amazonaws.com/powersOfTau28_hez_final_08.ptau

```
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


### 2. View information about the circuit
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

### 3. Calculate the witness

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

### 4. Setup

Currently, snarkjs supports 2 proving systems: groth16 and PLONK. 

Groth16 requires a trusted ceremony for each circuit. PLONK does not require it, it's enough with the powers of tau ceremony which is universal.
So, we'll be using PLONK for this demo
```sh
snarkjs plonk setup circuit.r1cs powersOfTau28_hez_final_08.ptau circuit_final.zkey
```

### 23. Create the proof

```sh
snarkjs plonk prove circuit_final.zkey witness.wtns proof.json public.json
```

### 24. Verify the proof

#### PLONK
```sh
snarkjs plonk verify verification_key.json public.json proof.json
```

### 25. Turn the verifier into a smart contract
```sh
snarkjs zkey export solidityverifier circuit_final.zkey verifier.sol
```

Finally, we export the verifier as a Solidity smart-contract so that we can publish it on-chain -- using [remix](https://remix.ethereum.org/) for example. For the details on how to do this, refer to section 4 of [this tutorial](https://blog.iden3.io/first-zk-proof.html).

### 26. Simulate a verification call
```sh
snarkjs zkey export soliditycalldata public.json proof.json
```

We use `soliditycalldata` to simulate a verification call, and cut and paste the result directly in the verifyProof field in the deployed smart contract in the remix environment.

And voila! That's all there is to it :)