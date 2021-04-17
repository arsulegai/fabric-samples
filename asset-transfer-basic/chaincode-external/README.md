# Asset-Transfer-Basic as an external service

This sample provides an introduction to how to use external builder and launcher scripts to run chaincode as an external service to your peer. For more information, see the [Chaincode as an external service](https://hyperledger-fabric.readthedocs.io/en/latest/cc_launcher.html) topic in the Fabric documentation.

**Note:** each organization in a real network would need to setup and host their own instance of the external service. For simplification purpose, in this sample we use the same instance for both organizations.

## Setting up the external builder and launcher

External Builders and Launchers is an advanced feature that typically requires custom packaging of the peer image so that it contains all the tools your builder and launcher require. For this sample we use very simple (and crude) shell scripts that can be run directly within the default Fabric peer images.

Open the `config/core.yaml` file at the top of the `fabric-samples` hierarchy. Note that this file comes along with the Fabric binaries, so if you don't have it, follow the [Install the Samples, Binaries and Docker Images](https://hyperledger-fabric.readthedocs.io/en/latest/install.html) instructions in the Hyperledger Fabric documentation to download the binaries and config files.

Modify the field `externalBuilders` as the following:
```
externalBuilders:
    - path: /opt/gopath/src/github.com/hyperledger/fabric-samples/asset-transfer-basic/chaincode-external/sampleBuilder
      name: external-sample-builder
```
This configuration sets the name of the external builder as `external-sample-builder`, and the path of the builder to the scripts provided in this sample. Note that this is the path within the peer container, not your local machine.

To set the path within the peer container, you will need to modify the container compose file to mount a couple of additional volumes.
Open the file `test-network/docker/docker-compose-test-net.yaml`, and add to the `volumes` section of both `peer0.org1.example.com` and `peer0.org2.example.com` the following two lines:

```
        - ../..:/opt/gopath/src/github.com/hyperledger/fabric-samples
        - ../../config/core.yaml:/etc/hyperledger/fabric/core.yaml
```

This will mount the fabric-sample builder into the peer container so that it can be found at the location specified in the config file,
and override the peer's core.yaml config file within the fabric-peer image so that the config file modified above is used.

## Packaging and installing Chaincode

The Asset-Transfer-Basic external chaincode requires two environment variables to run, `CHAINCODE_SERVER_ADDRESS` and `CHAINCODE_ID`, which are described and set in the `chaincode.env` file.

The peer needs a corresponding `connection.json` configuration file so that it can connect to the external Asset-Transfer-Basic service.

The address specified in the `connection.json` must correspond to the `CHAINCODE_SERVER_ADDRESS` value in `chaincode.env`, which is `asset-transfer-basic.org1.example.com:9999` in our example.

Because we will run our chaincode as an external service, the chaincode itself does not need to be included in the chaincode
package that gets installed to each peer. Only the configuration and metadata information needs to be included
in the package. Since the packaging is trivial, we can manually create the chaincode package.

First, create a `code.tar.gz` archive containing the `connection.json` file:

```
tar cfz code.tar.gz connection.json
```

Then, create the chaincode package, including the `code.tar.gz` file and the supplied `metadata.json` file:

```
tar cfz asset-transfer-basic-external.tgz metadata.json code.tar.gz
```

You are now ready to use the external chaincode. We will use the `test-network` sample to get a network setup and make use of it.

## Starting the test network

In a different terminal, from the `test-network` sample directory starts the network using the following command:

```
./network.sh up createChannel -c mychannel -ca
```

This starts the test network and creates the channel. We will now proceed to installing our external chaincode package.

## Installing the external chaincode

We can't use the `test-network/network.sh` script to install our external chaincode so we will have to do a bit more work by hand but we can still leverage part of the test-network scripts to make this easier.

First, get the functions to setup your environment as needed by running the following command (this assumes you are still in the `test-network` directory):

```
. ./scripts/envVar.sh
```

Install the `asset-transfer-basic-external.tar.gz` chaincode on org1:

```
setGlobals 1
../bin/peer lifecycle chaincode install ../asset-transfer-basic/chaincode-external/asset-transfer-basic-external.tgz
```

setGlobals simply defines a bunch of environment variables suitable to act as one organization or another, org1 or org2.

Install it on org2:

```
setGlobals 2
../bin/peer lifecycle chaincode install ../asset-transfer-basic/chaincode-external/asset-transfer-basic-external.tgz
```

This will output the chaincode pakage identifier such as `basic_1.0:0262396ccaffaa2174bc09f750f742319c4f14d60b16334d2c8921b6842c090c` that you will need to use in the following commands.

For convenience it is best to store your package id value in an environment variable so that it can be referenced in later commands:

```
export PKGID=basic_1.0:0262396ccaffaa2174bc09f750f742319c4f14d60b16334d2c8921b6842c090
```

If needed, you can query the installed chaincode to get its package-id:

```
setGlobals 1
../bin/peer lifecycle chaincode queryinstalled --peerAddresses localhost:7051 --tlsRootCertFiles organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
```

Edit the `chaincode.env` file in the `fabric-samples/asset-transfer-basic/chaincode-external` directory as necessary to set the `CHAINCODE_ID` variable to the chaincode package-id obtained above.


## Running the Asset-Transfer-Basic external service

To run the service in a container, from a different terminal, build an Asset-Transfer-Basic docker image, using the supplied `Dockerfile`, using the following command in the `fabric-samples/asset-transfer-basic/chaincode-external` directory:

```
docker build -t hyperledger/asset-transfer-basic .
```

Then, start the Asset-Transfer-Basic service:

```
docker run -it --rm --name asset-transfer-basic.org1.example.com --hostname asset-transfer-basic.org1.example.com --env-file chaincode.env --network=net_test hyperledger/asset-transfer-basic
```

This will start the container and start the external chaincode service within it.

## Finish deploying the Asset-Transfer-Basic external chaincode

Finishing the deployment of the chaincode on the test network can be done from the terminal you started the network from with the following commands (make sure the package-id is set to the value you received above):

```
setGlobals 2
../bin/peer lifecycle chaincode approveformyorg -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile $PWD/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --channelID mychannel --name basic --version 1.0 --package-id $PKGID --sequence 1

setGlobals 1
../bin/peer lifecycle chaincode approveformyorg -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile $PWD/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --channelID mychannel --name basic --version 1.0 --package-id $PKGID --sequence 1

../bin/peer lifecycle chaincode commit -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls --cafile $PWD/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --channelID mychannel --name basic --peerAddresses localhost:7051 --tlsRootCertFiles $PWD/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses localhost:9051 --tlsRootCertFiles organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt --version 1.0 --sequence 1
```

This approves the chaincode definition for both orgs and commits it using org1. This should result in an output similar to:

```
2020-08-05 15:41:44.982 PDT [chaincodeCmd] ClientWait -> INFO 001 txid [6bdbe040b99a45cc90a23ec21f02ea5da7be8b70590eb04ff3323ef77fdedfc7] committed with status (VALID) at localhost:7051
2020-08-05 15:41:44.983 PDT [chaincodeCmd] ClientWait -> INFO 002 txid [6bdbe040b99a45cc90a23ec21f02ea5da7be8b70590eb04ff3323ef77fdedfc7] committed with status (VALID) at localhost:9051
```

Now that the chaincode is deployed to the channel, and started as an external service, it can be used as normal.

## Using the Asset-Transfer-Basic external chaincode

From yet another terminal, go to `fabric-samples/asset-transfer-basic/application-javascript` and use the node application to test the chaincode you just installed with the following commands:

```
rm -rf wallet # in case you ran this before
npm install
node app.js
```

If all goes well, the program should run exactly the same as described in the "Writing Your First Application" tutorial.

## Enabling TLS for chaincode and peer communication

**Note:** This section is explained with an example of self-signed certificate. You may use organization hosted CA to issue the certificate and generate a key for production deployment.

In the example so far, you connected the single instance of chaincode server to both the peers in `test-network`. However, if you would like to verify TLS connection of the peer nodes from the chaincode, each peer node has its own root CA certificate. Enabling TLS is made possible at runtime in the chaincode.

- As a first step generate a keypair that can be used. Run these commands from the `fabric-samples/asset-transfer-basic/chaincode-external` directory.

*Find instructions to install `openssl` in [openssl.org](https://www.openssl.org/)*

For `org1.example.com`

```
openssl req -nodes -x509 -newkey rsa:4096 -keyout crypto/key1.pem -out crypto/cert1.pem -subj "/C=IN/ST=KA/L=Bangalore/O=example Inc/OU=Developer/CN=asset-transfer-basic.org1.example.com/emailAddress=dev@asset-transfer-basic.org1.example.com"
```

For `org2.example.com`

```
openssl req -nodes -x509 -newkey rsa:4096 -keyout crypto/key2.pem -out crypto/cert2.pem -subj "/C=IN/ST=KA/L=Bangalore/O=example Inc/OU=Developer/CN=asset-transfer-basic.org2.example.com/emailAddress=dev@asset-transfer-basic.org2.example.com"
```

- Copy the CA file contents for both `org1.example.com` & `org2.example.com`

```
cp ../../test-network/organizations/peerOrganizations/org1.example.com/ca/ca.org1.example.com-cert.pem crypto/rootcert1.pem
cp ../../test-network/organizations/peerOrganizations/org2.example.com/ca/ca.org2.example.com-cert.pem crypto/rootcert2.pem
```

- Generate a client key and cert for auth purpose. You need a key and cert generated from the CA of each organization. Peer nodes serve as clients to chaincode server.

- Change the `connection.json` with the below contents. `root_cert` parameter is the root CA certificate which the chaincode server is run with. You may run below commands to get the certificate file contents as string and copy them when needed.

```
awk 'NF {sub(/\r/, ""); printf "%s\\n",$0;}' crypto/cert1.pem
awk 'NF {sub(/\r/, ""); printf "%s\\n",$0;}' crypto/cert2.pem
```

Similarly, replace the `client_key` and the `client_cert` contents with values from the previous step.

```
{
  "address": "asset-transfer-basic.org1.example.com:9999",
  "dial_timeout": "10s",
  "tls_required": true,
  "client_auth_required": true,
  "client_key": "-----BEGIN PRIVATE KEY----- ... -----END PRIVATE KEY-----",
  "client_cert": "-----BEGIN CERTIFICATE---- ... -----END CERTIFICATE-----",
  "root_cert": "-----BEGIN CERTIFICATE---- ... -----END CERTIFICATE-----"
}
```

- Follow the instructions in [Package](#packaging-and-installing-chaincode) and [Install](#installing-the-external-chaincode) steps for each organization. Remember that chaincode server address for second organization is `asset-transfer-basic.org2.example.com:9999`.

- Copy the appropriate `CHAINCODE_ID` to both [chaincode1.env](./chaincode1.env) and [chaincode2.env](./chaincode2.env) files. Bring up chaincode containers using the docker-compose command below

```
docker-compose up -f docker-compose-chaincode.yaml up --build -d
```

- Follow the instructions in [Finish Deployment](#finish-deploying-the-asset-transfer-basic-external-chaincode-) for each organization seperately.
