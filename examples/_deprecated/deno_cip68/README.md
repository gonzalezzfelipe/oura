# CIP68 Parser Example

This example shows how to leverage the Deno filter stage to apply custom parsing logic and extract CIP68 reference NFT data from transaction without the need to change Oura's internal processing pipeline.

## Configuration

The relevant section of the daemon.toml is the following:

```toml
[[filters]]
type = "Deno"
main_module = "./parser.js"
use_async = false
```

The above configuration fragment instructs _Oura_ to introduce a _Deno_ filter that uses the logic specified in the file `parser.js`, which holds your custom filter logic.

The following section explains how to create a .js file compatible with what the Deno filter is expecting.

## Custom Deno Filter

To create the custom logic for your Deno-base filter, you need to start by creating a _Typescript_ and implementing a `mapEvent` function:

```ts
export function mapEvent(record: oura.Event) {
  // your custom logic goes here
}
```

The above function will be called for each record ths goes through _Oura_'s pipeline. Depending on your configuration, records could represent blocks, transactions or other payloads generated by previous filter stages.

## CIP68

The goal for this particular example is to extract data from transactions that corresponds to reference NFT as defined by [CIP68](https://cips.cardano.org/cips/cip68).

By inspecting the outputs and Plutus datums from the transactions, we can parse relevant information and generate custom JSON objects such as this one:

```json
{
    "label": "000643b042756438363031",
    "policy": "4523c5e21d409b81c95b45b0aea275b8ea1406e6cafea5583b9f8a5f",
    "metadata": {
        "name": "SpaceBud #8601",
        "traits": "",
        "type": "Shark",
        "image": "ipfs://bafkreidrqwxpxhyc5bo364fzzwv7nhnjel6y6zkywriw33jopb2p4tba5u",
        "sha256": "7185aefb9f02e85dbf70b9cdabf69da922fd8f6558b4516ded2e7874fe4c20ed"
    },
    "version": 1,
    "txHash": "720dd358d2b28531e181f93eed0e0d24db364232ddaccf6655abf92790a062d5"
}
```

You can check the actual code for the parser in the file `parser.ts`. Without going much into detail, the relevant part of the parser is a function called `extractRefNFT`:

```ts
function extractNFTRef(
  output: oura.TxOutputRecord,
  allDatums?: oura.PlutusDatumRecord[] | null
): Partial<RefNFT> | null {
  const asset = output.assets?.find((a) => a.asset.startsWith("000"));
  
  if (!asset) return null;

  const datum =
    output.inline_datum ||
    allDatums?.find((d) => d.datum_hash == output.datum_hash);

  if (!datum) return null;

  return {
    label: asset.asset,
    policy: asset.policy,
    ...parseDatum(datum),
  };
}
```

## Typescript Bundle

Before being able to use our custom code in Oura, we need to instruct _Deno_ to "bundle" it. This process will resolve all dependencies (local or remote), transpile Typescript into Javascript and generate a single `.js` file which is ready to be used by _Oura_'s pipeline.

Run the following shell script to bundle our custom code:

```sh
deno bundle parser.ts parser.js
```

Assuming there aren't any errors in the code, a new `parser.js` should be generated in the same folder.

## Run Oura

We're now ready to run our Oura pipeline to get CIP68 reference NFT records. Assuming you have _Oura_ `v2` installed on your system, run the following shell script from the `examples/deno_cip68` folder to start the pipeline process:

```sh
cargo run --bin oura -- daemon --config ./daemon.toml
```

After a while, you should start seeing JSON objects printed out through stdout.