# General Attestation Interface

Support Gramine, Occlum and SGX SDK.
```mermaid
classDiagram
    AttestationCLI ..> AttestationService
    AttestationCLI ..> QuoteGenerator
    AttestationCLI ..> QuoteVerifier
    AttestationCLI: +appID
    AttestationCLI: +apiKey
    AttestationCLI: +asType
    AttestationCLI: +asURL
    AttestationCLI: +challenge
    AttestationCLI: +policyID
    AttestationCLI: +userReport

    AttestationService <|-- EHSMAttestationService 
    AttestationService <|-- DummyAttestationService
    AttestationService <|-- DCAPAttestationService
    AttestationService: +register()
    AttestationService: +getQuoteFromServer(challenge)
    AttestationService: +attWithServer(quote, policyID)
        EHSMAttestationService: +kmsServerIP
        EHSMAttestationService: +kmsServerPort
        EHSMAttestationService: +ehsmAPPID
        EHSMAttestationService: +ehsmAPIKEY
        EHSMAttestationService: +payLoad
        EHSMAttestationService: +contrustUrl()
        EHSMAttestationService: +getQuoteFromServer(challenge)
        EHSMAttestationService: +attWithServer(quote, policyID)

        DummyAttestationService: +getQuoteFromServer(challenge)
        DummyAttestationService: +attWithServer(quote, policyID)

        DCAPAttestationService: +getQuoteFromServer(challenge)
        DCAPAttestationService: +attWithServer(quote, policyID)
    QuoteGenerator <|-- GramineQuoteGeneratorImpl
    QuoteGenerator: +getQuote(userReport)
        GramineQuoteGeneratorImpl: +getQuote(userReport)
    QuoteVerifier <|-- SGXDCAPQuoteVerifierImpl
    QuoteVerifier: +verifyQuote(asQuote)
        SGXDCAPQuoteVerifierImpl: +verifyQuote(asQuote)
```
## Environment
You should have an available attestation service to attest with. You can use `EHSMAttestationService` and configure eHSM-KMS according to [this link](https://github.com/intel-analytics/BigDL/tree/main/ppml/services/ehsm/kubernetes), or you can just use `DummyAttestationService` for debug. 

### Bidirectional Attestation
To enable bidirectional attestation, you also need SGX SDK to fulfill quote verification. Here is the guide to install SGX SDK with related libs. 

```bash
wget https://download.01.org/intel-sgx/sgx-linux/2.16/as.ld.objdump.r4.tar.gz 
tar -zxf as.ld.objdump.r4.tar.gz
sudo cp external/toolset/ubuntu20.04/* /usr/local/bin

wget https://download.01.org/intel-sgx/sgx-dcap/1.13/linux/distro/ubuntu20.04-server/sgx_linux_x64_sdk_2.16.100.4.bin
#choose to install the sdk into the /opt/intel
chmod a+x ./sgx_linux_x64_sdk_2.16.100.4.bin && sudo ./sgx_linux_x64_sdk_2.16.100.4.bin

source /opt/intel/sgxsdk/environment

cd /opt/intel

wget https://download.01.org/intel-sgx/sgx-dcap/1.13/linux/distro/ubuntu20.04-server/sgx_debian_local_repo.tgz

tar xzf sgx_debian_local_repo.tgz

echo 'deb [trusted=yes arch=amd64] file:///opt/intel/sgx_debian_local_repo focal main' | tee /etc/apt/sources.list.d/intel-sgx.list

wget -qO - https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key | apt-key add -

sudo apt-get update

sudo apt-get install -y libsgx-enclave-common-dev  libsgx-ae-qe3 libsgx-ae-qve libsgx-urts libsgx-dcap-ql libsgx-dcap-default-qpl libsgx-dcap-quote-verify-dev libsgx-dcap-ql-dev libsgx-dcap-default-qpl-dev libsgx-quote-ex-dev libsgx-uae-service libsgx-ra-network libsgx-ra-uefi
```

And you need to configure PCCS in `/etc/sgx_default_qcnl.conf`.

```bash
# PCCS server address
PCCS_URL=https://your_pccs_url/sgx/certification/v3/

# To accept insecure HTTPS certificate, set this option to FALSE
USE_SECURE_CERT=FALSE
```

## Usage
You can attest your environment with AttestationCLI by command like:
```bash
java -cp [dependent-jars] com.intel.analytics.bigdl.ppml.attestation.AttestationCLI -i <appID> -k <apiKey> -u <asURL> -t <asType> -c <challenge> -p <userReport> 
```

## Parameters
`-i` **appID** , `-k` **apiKey** The appID and apipey pair generated by your attestation service. 

`-u` **asURL** URL of attestation service. Should match the format `<ip_address>:<port>`, default is `127.0.0.1:9000`

`-t` **asType** Type of attestation service. Currently support `DummyAttestationService` and `EHSMAttestationService`, default is `EHSMAttestationService`.

`-c` **challenge** Challenge to get quote of attestation service which will be verified by local SGX SDK. Used only for bi-attestation. Should be a BASE64 string, default is "" and will skip bi-attestation.

`-p` **userReport** User report to generate quote and attested by attestation service. Default is `test`.

# Attestation Service Verification Interface

You can verify Attestation Service (eHSM for example) with VerificationCLI. It will first get quote from Attestation Service and then verify the quote with SGX SDK.

## Environment 
To verify SGX quote, you can follow [this guide](#bi-attestation) to install SGX SDK and related DCAP libraries. For TDX quote, you can refer [this part](#tdx-quote-verification-interface) to install dependent components.
## Usage
You can attest the attestation service with VerificationCLI by command like:
```bash
java -cp [dependent-jars] com.intel.analytics.bigdl.ppml.attestation.VerificationCLI -i <appID> -k <apiKey> -u <asURL> -t <asType> -c <challenge> -q <quotePath>
```
Or you can use `verify-attestation-service.sh` to verify the attestation service quote.
```bash
cd ../../../../../../../../../../..
cd ppml/trusted-big-data-ml/python/docker-gramine/base/
bash verify-attestation-service.sh
```

## Parameters
`-i` **appID** , `-k` **apiKey** The appID and apiKey pair generated by your attestation service. 

`-u` **asURL** URL of attestation service. Should match the format `<ip_address>:<port>`, default is `127.0.0.1:9000`

`-t` **asType** Type of attestation service. Currently support `EHSMAttestationService`.

`-c` **challenge** Challenge to get quote of attestation service which will be verified by local SGX SDK. Should be a BASE64 string.

`-q` **quotePath** Only set to verify local quote. Will **disable** getting quote from attestation service.

# TDX Quote Generation Interface

You can generate TDX quote in TDVM with `TDXQuoteGenerate`.

## Requirements
* TDVM

Check whether the device `/dev/tdx-attest` exists.

* Intel SGX SDK
* Intel SGX DCAP Development Packages
  
Install with commands:
```bash ubuntu 20.04
# install sgxsdk
cd /opt/intel && \
wget https://download.01.org/intel-sgx/sgx-dcap/1.14/linux/distro/ubuntu20.04-server/sgx_linux_x64_sdk_2.17.100.3.bin && \
chmod a+x ./sgx_linux_x64_sdk_2.17.100.3.bin && \
printf "no\n/opt/intel\n"|./sgx_linux_x64_sdk_2.17.100.3.bin && \
. /opt/intel/sgxsdk/environment && \
# install dcap
cd /opt/intel && \
wget https://download.01.org/intel-sgx/sgx-dcap/1.14/linux/distro/ubuntu20.04-server/sgx_debian_local_repo.tgz && \
tar xzf sgx_debian_local_repo.tgz && \
echo 'deb [trusted=yes arch=amd64] file:///opt/intel/sgx_debian_local_repo focal main' | tee /etc/apt/sources.list.d/intel-sgx.list && \
wget -qO - https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key | apt-key add - && \
apt-get update && \
# TODO: minimize lib dependency
apt-get install -y libsgx-enclave-common-dev libsgx-ae-qe3 libsgx-ae-qve libsgx-urts libsgx-dcap-ql libsgx-dcap-default-qpl libsgx-dcap-quote-verify-dev libsgx-dcap-ql-dev libsgx-dcap-default-qpl-dev libsgx-quote-ex-dev libsgx-uae-service libsgx-ra-network libsgx-ra-uefi libtdx-attest libtdx-attest-dev
```

* TDX PCCS
  
You can deploy a PCCS service container with [this](https://github.com/intel-analytics/BigDL/tree/main/ppml/services/pccs/docker). Modify `uri` and `api_key` in `default.json`.
```
    "uri": "https://sbx.api.trustedservices.intel.com/sgx/certification/v4/",
    "ApiKey": "your_subscription_key",
```

## Usage
```bash
java -cp [dependent-jars] com.intel.analytics.bigdl.ppml.attestation.TdxQuoteGenerate -r <userReport>
```

## Parameters
`-r` **userReport** User report data which will be passed to quote.

# TDX Quote Verification Interface

You can verify TDX quote with `VerificationCLI`.

## Requirements
* Intel SGX SDK
* Intel SGX DCAP Development Packages
  
Install with commands:
```bash ubuntu 20.04
# install sgxsdk
cd /opt/intel && \
wget https://download.01.org/intel-sgx/sgx-dcap/1.14/linux/distro/ubuntu20.04-server/sgx_linux_x64_sdk_2.17.100.3.bin && \
chmod a+x ./sgx_linux_x64_sdk_2.17.100.3.bin && \
printf "no\n/opt/intel\n"|./sgx_linux_x64_sdk_2.17.100.3.bin && \
. /opt/intel/sgxsdk/environment && \
# install dcap
cd /opt/intel && \
wget https://download.01.org/intel-sgx/sgx-dcap/1.14/linux/distro/ubuntu20.04-server/sgx_debian_local_repo.tgz && \
tar xzf sgx_debian_local_repo.tgz && \
echo 'deb [trusted=yes arch=amd64] file:///opt/intel/sgx_debian_local_repo focal main' | tee /etc/apt/sources.list.d/intel-sgx.list && \
wget -qO - https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key | apt-key add - && \
apt-get update && \
apt-get install -y libsgx-enclave-common-dev libsgx-ae-qe3 libsgx-ae-qve libsgx-urts libsgx-dcap-ql libsgx-dcap-default-qpl libsgx-dcap-quote-verify-dev libsgx-dcap-ql-dev libsgx-dcap-default-qpl-dev libsgx-quote-ex-dev libsgx-uae-service libsgx-ra-network libsgx-ra-uefi
```
* TDX quote verification lib

Apply `libsgx_dcap_quoteverify.so` from Intel S3 team, use minor version accordingly. (`1.12.100.3` for example)
```
chmod +x libsgx_dcap_quoteverify.so
sudo cp libsgx_dcap_quoteverify.so /usr/lib/x86_64-linux-gnu/libsgx_dcap_quoteverify.so.1.12.100.3
```

* TDX PCCS
  
You can deploy a PCCS service container with [this](https://github.com/intel-analytics/BigDL/tree/main/ppml/services/pccs/docker). Modify `uri` and `api_key` in `default.json`.
```
    "uri": "https://sbx.api.trustedservices.intel.com/sgx/certification/v4/",
    "ApiKey": "your_subscription_key",
```

## Usage
```bash
java -cp [dependent-jars] com.intel.analytics.bigdl.ppml.attestation.VerificationCLI -q <quotePath>
```

## Parameters
`-q` **quotePath** Path of quote to be verified.