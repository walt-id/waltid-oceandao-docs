## 🚨 Repository Moved

> [!IMPORTANT]
> This repository is no longer maintained.
> 
> All walt.id development continues in the monorepo:
> 
### 👉 Code **https://github.com/walt-id/waltid-identity** & Docs **https://docs.walt.id**

# Instructions for the OceanDAO SSI Use Case

This guide demonstrates how to create a Decentralized Identifier (DID) and to register it on the EBSI ledger. Furthermore the issuance as well as the verification-process of the _GaiaxCredential_ is shown.

## Setting up walt.id SSI Kit

### 1. Initiating directory with the walt.id SSI Kit
In the after-next step, we will use docker to create a container mount, which will be owned by root:root. To not have random root-owned directories laying around, we recommend first creating a new directory.

    mkdir issuer-service && cd issuer-service

Pull the container:

    docker pull docker.io/waltid/ssikit

### 2. Create an alias

We will continue to create an alias, otherwise the commands will be incredibly long. Of course this isn't actually mandatory, you can easily replace all instances of the alias-command "ssikit" with the content between the quotes in the next line.

    alias ssikit="docker container run -p 7000-7004:7000-7004 -itv $(pwd)/data:/app/data docker.io/waltid/ssikit"

Now all sensitive data, like cryptographic key, DIDs and credentials will be stored with root-rights in the "data" directory.  
Try it out:

    ssikit -h


>walt.id SSI Kit 1.1-SNAPSHOT (running on Java 16.0.2+7-67) 
>  
>  
>  SSI Kit by walt.id  
>  
>  ...  

### 3. Running as REST service

    ssikit serve             # binds to 127.0.0.1

    ssikit serve -b 0.0.0.0  # binds to 0.0.0.0 <--- needed for docker

>  walt.id Core API:      http://127.0.0.1:7000  
>  walt.id Signatory API: http://127.0.0.1:7001  
>  walt.id Custodian API: http://127.0.0.1:7002  
>  walt.id Auditor API:   http://127.0.0.1:7003  
>  walt.id ESSIF API:     http://127.0.0.1:7004


##  Setting up an Issuer on the EBSI ledger

### 1. Key Creation
We start out by generating a Secp256k1 key.

**_CLI_**

    ssikit key gen -a Secp256k1

> Generating Secp256k1 key pair...  
> Key "efc581186a0945d8af75cdb8e1b16033" generated.
> 

_**API**_

curl -X POST "https://core.ssikit.walt.id/v1/key/gen" -H  "accept: application/json" -H  "Content-Type: application/json" -d "{\"keyAlgorithm\":\"ECDSA_Secp256k1\"}"
>{
>   "id": "38227a5b9f104a72be6b605d8da7544d"
>}
> 
### 2. DID EBSI creation
We continue to create a **did:ebsi** using the key which we've just generated.

**_CLI_**

    ssikit did create -m ebsi -k efc581186a0945d8af75cdb8e1b16033
    # obviously we'll use the key here that we've just generated

> DID created: did:ebsi:zyc8qqkQifbyqZF7GGHW8zS  
> DID document (below, JSON): ...

_**API**_

curl -X POST "https://core.ssikit.walt.id/v1/did/create" -H  "accept: application/json" -H  "Content-Type: application/json" -d "{\"method\":\"ebsi\",\"keyAlias\":\"38227a5b9f104a72be6b605d8da7544d\"}"

> did:ebsi:zrgUxYjGLSBz2tYW75RUtbt

### 3. Registering DID EBSI on the ledger

To use the EBSI API, we first have to get our EBSI bearer token.

On https://app.preprod.ebsi.eu/users-onboarding click "Onboard with Captcha" if you don't have an EU account. Choose "Desktop Wallet" to have the session token displayed (Mobile Wallet will show the token encoded in a QR code).

Save your bearer-token somewhere the container is able to access, e.g. `data/bearer-token.txt`. Depending on your system, you'll have to write there as `root` user.

    echo "eyJhbGc_YOUR_TOKEN_HERE..." | sudo tee data/bearer-token.txt

#### 3.1. Onboarding flow:
Enter the **did:ebsi** that was generated above and supply the bearer-token.

**_CLI_**

    ssikit essif onboard --did did:ebsi:zyc8qqkQifbyqZF7GGHW8zS data/bearer-token.txt

> ESSIF onboarding for DID did:ebsi:zyc8qqkQifbyqZF7GGHW8zS running...  
> ESSIF onboarding for DID did:ebsi:zyc8qqkQifbyqZF7GGHW8zS was performed successfully.

curl -X POST "https://essif.ssikit.walt.id/v1/client/onboard" -H  "accept: application/json" -H  "Content-Type: application/json" -d "{\"bearerToken\":\"eyJhbGciOiJF...\",\"did\":\"did:ebsi:zrgUxYjGLSBz2tYW75RUtbt\"}"

_**API**_

>"{\"verifiableCredential\":{\"id\":\"vc:ebsi:authentication#1e36b26b-abb0-4bc ..."

#### 3.2 Authentication API flow:

**_CLI_**

    ssikit essif auth-api --did did:ebsi:zyc8qqkQifbyqZF7GGHW8zS

> EBSI Authentication API flow for DID did:ebsi:zyc8qqkQifbyqZF7GGHW8zS running...  
> EBSI Authentication API flow for DID did:ebsi:zyc8qqkQifbyqZF7GGHW8zS was performed successfully.  

_**API**_

    curl -X POST "https://essif.ssikit.walt.id/v1/client/auth" -H  "accept: application/json" -H  "Content-Type: text/plain" -d "did:ebsi:zrgUxYjGLSBz2tYW75RUtbt"

> {}

#### 3.3 Writing to ledger (signing of ETH transaction):

**_CLI_**

    ssikit essif did register --did did:ebsi:zyc8qqkQifbyqZF7GGHW8zS  

> EBSI ledger registration for DID did:ebsi:zyc8qqkQifbyqZF7GGHW8zS running...  
> EBSI ledger registration for DID did:ebsi:zyc8qqkQifbyqZF7GGHW8zS was performed successfully.  
> Call command: 'did resolve --did did:ebsi:zyc8qqkQifbyqZF7GGHW8zS' in order to retrieve the DID document from the EBSI ledger.  

_**API**_

    curl -X POST "https://essif.ssikit.walt.id/v1/client/registerDid" -H  "accept: application/json" -H  "Content-Type: text/plain" -d "did:ebsi:zrgUxYjGLSBz2tYW75RUtbt"

> {}
#### 3.4 Try out DID resolving via CLI:

_**CLI**_

    ssikit did resolve --did did:ebsi:zyc8qqkQifbyqZF7GGHW8zS

> Resolving DID "did:ebsi:zyc8qqkQifbyqZF7GGHW8zS"...  
>  
> Results:  
>  
> DID resolved: "did:ebsi:zyc8qqkQifbyqZF7GGHW8zS"  
> DID document (below, JSON):  
> ...

_**API**_

    curl -X POST "https://core.ssikit.walt.id/v1/did/resolve" -H  "accept: application/json" -H  "Content-Type: application/json" -d "{\"did\":\"did:ebsi:zrgUxYjGLSBz2tYW75RUtbt\"}"

    {
        "authentication":[
           "did:ebsi:zrgUxYjGLSBz2tYW75RUtbt#38227a5b9f104a72be6b605d8da7544d"
        ],
       "@context":[
           "https://www.w3.org/ns/did/v1"
        ],
        "id":"did:ebsi:zrgUxYjGLSBz2tYW75RUtbt",
        "verificationMethod":[
            {
            "controller":"did:ebsi:zrgUxYjGLSBz2tYW75RUtbt",
            "id":"did:ebsi:zrgUxYjGLSBz2tYW75RUtbt#38227a5b9f104a72be6b605d8da7544d",
            "publicKeyJwk":{
                "alg":"ES256K",
                "crv":"secp256k1",
                "kid":"38227a5b9f104a72be6b605d8da7544d",
                "kty":"EC",
                "use":"sig",
                "x":"-PYAGskggDHO5-LGprwtKG0uB-KObZDb40ZTH_3Ag1E",
                "y":"Z0uHycCZW0dWUlHOyW8BDS3ZV3UOFjEvFsSmtT1KJ-k"
            },
            "type":"Secp256k1VerificationKey2018"
            }
        ]
    }

Alternatively, the DID can also be resolved directly using the ESSIF API:

Their swagger interface: https://api.preprod.ebsi.eu/docs/?urls.primaryName=DID%20Registry%20API#/DID%20Registry/get-did-registry-v2-identifier

## 4. Issuing a custom credential for the deltaDAO Datamarketplace

We'll issue a custom credential based on the schema deltaDAO provided, the issuer being a subject with a did:ebsi key (the one we've just created and registered to EBSI above), and the holder having another key format, we'll use a normal did:key for this demo. You may also mix in other DIDs (e.g. for the verifier), like did:web.

### 4.1. Creating a DID for the recipient (holder) of the credential 
Business as usual:

    ssikit did create -m key

    curl -X POST "https://core.ssikit.walt.id/v1/did/create" -H  "accept: application/json" -H  "Content-Type: application/json" -d "{\"method\":\"key\"}"

### 4.2. Issuing W3C Verifiable Credential based on the deltaDAO credential schema (with the CLI)
To try this example with the REST API, see "Via REST (Swagger)".

We'll use the interactive CLI data provider for this template.

_**CLI**_

    ssikit vc issue -t GaiaxCredential -i did:ebsi:zyc8qqkQifbyqZF7GGHW8zS -s did:key:z6MkwgKrgtL9mvxJLikqsXUNPxSko17bPb5fxvvurcditnX6 --interactive data/vc.json

You'll be able to specify the different fields in the `credentialSubject` using the CLI-DataProvider (`--interactive`), you can also test out the normal DataProvider here. The credential will be saved to `data/vc.json` (as specified).

> Issuing a verifiable credential (using template GaiaxCredential)  
>  
> - Subject information  
>  
> Legally binding name [deltaDAO AG]: SSI Fabric GmbH  
> Brand name [deltaDAO]: Walt.ID  
> Legal registration number [HRB 170364]:  
> Corporate email address [contact@delta-dao.com]: ...  

(By pressing enter, you will accept the default, which is described in the brackets "Name [default]").

> Generated Credential: ...  
> Saved credential to file: data/vc.json

_**API**_

    curl -X POST "https://signatory.ssikit.walt.id/v1/credentials/issue" -H  "accept: application/json" -H  "Content-Type: application/json" -d "{\"templateId\":\"GaiaxCredential\",\"config\":{\"issuerDid\":\"did:ebsi:zrgUxYjGLSBz2tYW75RUtbt\",\"subjectDid\":\"did:key:z6MkrtMb2FYe6Wqvw8CXUWSQfuieaDBsHY4fhrchmpVw7zYW\",\"proofType\":\"LD_PROOF\"}}"

    {
        "@context" : [ "https://www.w3.org/2018/credentials/v1" ],
        "credentialSubject" : {
            "DNSpublicKey" : "04:8B:CA:33:B1:A1:3A:69:E6:A2:1E:BE:CB:4E:DF:75:A9:70:8B:AA:51:83:AB:A1:B0:5A:35:20:3D:B4:29:09:AD:67:B4:12:19:3B:6A:B5:7C:12:3D:C4:CA:DD:A5:E0:DA:05:1E:5E:1A:4B:D1:F2:BA:8F:07:4D:C7:B6:AA:23:46",
            "brandName" : "deltaDAO",
            "commercialRegister" : {
                "countryName" : "Germany",
                "locality" : "Hamburg",
                "organizationName" : "Amtsgericht Hamburg (-Mitte)",
                "organizationUnit" : "Registergericht",
                "postalCode" : "20355",
                "streetAddress" : "Caffamacherreihe 20"
            },
            "corporateEmailAddress" : "contact@delta-dao.com",
            "ethereumAddress" : {
                "id" : "0x4C84a36fCDb7Bc750294A7f3B5ad5CA8F74C4A52"
            },
            "id" : "did:key:z6MkrtMb2FYe6Wqvw8CXUWSQfuieaDBsHY4fhrchmpVw7zYW",
         ...

### 4.3. Creating a W3C Verifiable Presentation

Using the credential we've just generated, we will generate a Verifiable Presentation (VP) as a holder (with the did:key).

_**CLI**_

    ssikit vc present --holder-did did:key:z6MkwgKrgtL9mvxJLikqsXUNPxSko17bPb5fxvvurcditnX6 data/vc.json

> Creating a verifiable presentation for DID "did:key:z6MkwgKrgtL9mvxJLikqsXUNPxSko17bPb5fxvvurcditnX6"...  
> Using 1 VC:  
> - 1. data/vc.json (GaiaxCredential)  
>  
> Results:  
>  
> Verifiable presentation generated for holder DID: "did:key:z6MkwgKrgtL9mvxJLikqsXUNPxSko17bPb5fxvvurcditnX6"
> Verifiable presentation document (below, JSON):
> ...
> Verifiable presentation was saved to file: "data/vc/presented/vp-1635188200959.json"  

_**API**_


    curl -X POST "http://localhost:7002/credentials/present" -H  "accept: application/json" -H  "Content-Type: application/json" -d "{\"vcs\":[\"{\\\"@context\\\" : [\\\"https://www.w3.org/2018/credentials/v1\\\"], \\\"credentialSchema\\\" : {\\\"id\\\" : \\\"https://api.preprod.ebsi.eu/trusted-schemas-registry/v1/schemas/0xbf78fc08a7a9f28f5479f58dea269d3657f54f13ca37d380cd4e92237fb691dd\\\", \\\"type\\\" : \\\"JsonSchemaValidator2018\\\"}, \\\"credentialStatus\\\" : {\\\"id\\\" : \\\"https://essif.europa.eu/status/education#higherEducation#392ac7f6-399a-437b-a268-4691ead8f176\\\", \\\"type\\\" : \\\"CredentialStatusList2020\\\"}, \\\"credentialSubject\\\" : {\\\"awardingOpportunity\\\" : {\\\"awardingBody\\\" : {\\\"eidasLegalIdentifier\\\" : \\\"Unknown\\\", \\\"homepage\\\" : \\\"https://leaston.bcdiploma.com/\\\", \\\"id\\\" : \\\"did:ebsi:2A9BZ9SUe6BatacSpvs1V5CdjHvLpQ7bEsi2Jb6LdHKnQxaN\\\", \\\"preferredName\\\" : \\\"Leaston University\\\", \\\"registration\\\" : \\\"0597065J\\\"}, \\\"endedAtTime\\\" : \\\"2020-06-26T00:00:00Z\\\", \\\"id\\\" : \\\"https://leaston.bcdiploma.com/law-economics-management#AwardingOpportunity\\\", \\\"identifier\\\" : \\\"https://certificate-demo.bcdiploma.com/check/87ED2F2270E6C41456E94B86B9D9115B4E35BCCAD200A49B846592C14F79C86BV1Fnbllta0NZTnJkR3lDWlRmTDlSRUJEVFZISmNmYzJhUU5sZUJ5Z2FJSHpWbmZZ\\\", \\\"location\\\" : \\\"FRANCE\\\", \\\"startedAtTime\\\" : \\\"2019-09-02T00:00:00Z\\\"}, \\\"dateOfBirth\\\" : \\\"1993-04-08\\\", \\\"familyName\\\" : \\\"DOE\\\", \\\"givenNames\\\" : \\\"Jane\\\", \\\"gradingScheme\\\" : {\\\"id\\\" : \\\"https://leaston.bcdiploma.com/law-economics-management#GradingScheme\\\", \\\"title\\\" : \\\"Lower Second-Class Honours\\\"}, \\\"id\\\" : \\\"did:ebsi:2AEMAqXWKYMu1JHPAgGcga4dxu7ThgfgN95VyJBJGZbSJUtp\\\", \\\"identifier\\\" : \\\"0904008084H\\\", \\\"learningAchievement\\\" : {\\\"additionalNote\\\" : [\\\"DISTRIBUTION MANAGEMENT\\\"], \\\"description\\\" : \\\"MARKETING AND SALES\\\", \\\"id\\\" : \\\"https://leaston.bcdiploma.com/law-economics-management#LearningAchievment\\\", \\\"title\\\" : \\\"MASTERS LAW, ECONOMICS AND MANAGEMENT\\\"}, \\\"learningSpecification\\\" : {\\\"ectsCreditPoints\\\" : 120, \\\"eqfLevel\\\" : 7, \\\"id\\\" : \\\"https://leaston.bcdiploma.com/law-economics-management#LearningSpecification\\\", \\\"iscedfCode\\\" : [\\\"7\\\"], \\\"nqfLevel\\\" : [\\\"7\\\"]}}, \\\"evidence\\\" : {\\\"documentPresence\\\" : [\\\"Physical\\\"], \\\"evidenceDocument\\\" : [\\\"Passport\\\"], \\\"id\\\" : \\\"https://essif.europa.eu/tsr-va/evidence/f2aeec97-fc0d-42bf-8ca7-0548192d5678\\\", \\\"subjectPresence\\\" : \\\"Physical\\\", \\\"type\\\" : [\\\"DocumentVerification\\\"], \\\"verifier\\\" : \\\"did:ebsi:2962fb784df61baa267c8132497539f8c674b37c1244a7a\\\"}, \\\"expirationDate\\\" : \\\"2022-08-31T00:00:00Z\\\", \\\"id\\\" : \\\"education#higherEducation#392ac7f6-399a-437b-a268-4691ead8f176\\\", \\\"issuanceDate\\\" : \\\"2021-08-31T00:00:00Z\\\", \\\"issuer\\\" : \\\"did:ebsi:2A9BZ9SUe6BatacSpvs1V5CdjHvLpQ7bEsi2Jb6LdHKnQxaN\\\", \\\"validFrom\\\" : \\\"2021-08-31T00:00:00Z\\\", \\\"type\\\" : [\\\"VerifiableCredential\\\", \\\"VerifiableAttestation\\\", \\\"VerifiableDiploma\\\"]}\"],\"holderDid\":\"did:key:z6MkiiZXErau1mJa98sAD56irpa9oEgvPUNkMYoyhMfLjCAq\",\"verifierDid\":\"did:key:z6MkiiZXErau1mJa98sAD56irpa9oEgvPUNkMYoyhMfLjCAq\"}"

```json
{
  "@context" : [ "https://www.w3.org/2018/credentials/v1" ],
  "holder" : "did:key:z6MkiiZXErau1mJa98sAD56irpa9oEgvPUNkMYoyhMfLjCAq",
  "id" : "urn:uuid:1529ecbd-6a4e-406f-97ed-f628c16d63ef",
  "verifiableCredential" : [ {
    "@context" : [ "https://www.w3.org/2018/credentials/v1" ],
    "credentialSchema" : {
      "id" : "https://api.preprod.ebsi.eu/trusted-schemas-registry/v1/schemas/0xbf78fc08a7a9f28f5479f58dea269d3657f54f13ca37d380cd4e92237fb691dd",
      "type" : "JsonSchemaValidator2018"
    },
    "credentialStatus" : {
      "id" : "https://essif.europa.eu/status/education#higherEducation#392ac7f6-399a-437b-a268-4691ead8f176",
      "type" : "CredentialStatusList2020"
    },
    "credentialSubject" : {
      "awardingOpportunity" : {
        "awardingBody" : {
          "eidasLegalIdentifier" : "Unknown",
          "homepage" : "https://leaston.bcdiploma.com/",
          "id" : "did:ebsi:2A9BZ9SUe6BatacSpvs1V5CdjHvLpQ7bEsi2Jb6LdHKnQxaN",
          "preferredName" : "Leaston University",
          "registration" : "0597065J"
        },
        "endedAtTime" : "2020-06-26T00:00:00Z",
        "id" : "https://leaston.bcdiploma.com/law-economics-management#AwardingOpportunity",
        "identifier" : "https://certificate-demo.bcdiploma.com/check/87ED2F2270E6C41456E94B86B9D9115B4E35BCCAD200A49B846592C14F79C86BV1Fnbllta0NZTnJkR3lDWlRmTDlSRUJEVFZISmNmYzJhUU5sZUJ5Z2FJSHpWbmZZ",
        "location" : "FRANCE",
        "startedAtTime" : "2019-09-02T00:00:00Z"
      },
      "dateOfBirth" : "1993-04-08",
      "familyName" : "DOE",
      "givenNames" : "Jane",
      "gradingScheme" : {
        "id" : "https://leaston.bcdiploma.com/law-economics-management#GradingScheme",
        "title" : "Lower Second-Class Honours"
      },
      "id" : "did:ebsi:2AEMAqXWKYMu1JHPAgGcga4dxu7ThgfgN95VyJBJGZbSJUtp",
      "identifier" : "0904008084H",
      "learningAchievement" : {
        "additionalNote" : [ "DISTRIBUTION MANAGEMENT" ],
        "description" : "MARKETING AND SALES",
        "id" : "https://leaston.bcdiploma.com/law-economics-management#LearningAchievment",
        "title" : "MASTERS LAW, ECONOMICS AND MANAGEMENT"
      },
      "learningSpecification" : {
        "ectsCreditPoints" : 120,
        "eqfLevel" : 7,
        "id" : "https://leaston.bcdiploma.com/law-economics-management#LearningSpecification",
        "iscedfCode" : [ "7" ],
        "nqfLevel" : [ "7" ]
      }
    },
    "evidence" : {
      "documentPresence" : [ "Physical" ],
      "evidenceDocument" : [ "Passport" ],
      "id" : "https://essif.europa.eu/tsr-va/evidence/f2aeec97-fc0d-42bf-8ca7-0548192d5678",
      "subjectPresence" : "Physical",
      "type" : [ "DocumentVerification" ],
      "verifier" : "did:ebsi:2962fb784df61baa267c8132497539f8c674b37c1244a7a"
    },
    "expirationDate" : "2022-08-31T00:00:00Z",
    "id" : "education#higherEducation#392ac7f6-399a-437b-a268-4691ead8f176",
    "issuanceDate" : "2021-08-31T00:00:00Z",
    "issuer" : "did:ebsi:2A9BZ9SUe6BatacSpvs1V5CdjHvLpQ7bEsi2Jb6LdHKnQxaN",
    "validFrom" : "2021-08-31T00:00:00Z",
    "type" : [ "VerifiableCredential", "VerifiableAttestation", "VerifiableDiploma" ]
  } ],
  "type" : [ "VerifiablePresentation" ],
  "proof" : {
    "type" : "Ed25519Signature2018",
    "creator" : "did:key:z6MkiiZXErau1mJa98sAD56irpa9oEgvPUNkMYoyhMfLjCAq",
    "created" : "2021-12-02T22:26:45Z",
    "domain" : "https://api.preprod.ebsi.eu",
    "nonce" : "62dd4f6d-34c7-4604-84b2-86b4b93c5966",
    "proofPurpose" : "authentication",
    "verificationMethod" : "z6MkiiZXErau1mJa98sAD56irpa9oEgvPUNkMYoyhMfLjCAq#z6MkiiZXErau1mJa98sAD56irpa9oEgvPUNkMYoyhMfLjCAq",
    "jws" : "eyJiNjQiOmZhbHNlLCJjcml0IjpbImI2NCJdLCJhbGciOiJFZERTQSJ9..DxyqIYfdNsqD04zANrMstQp8IUbXE0o7h92hqQ7z7jbEOrbkTQqsHLOAM1hpscavx550hK6TCC975LWbxF3IBw"
  }
}
```

### 4.4. Verifying the VP

_**CLI**_

    ssikit vc verify data/vc/presented/vp-1635188200959.json -p SignaturePolicy -p JsonSchemaPolicy -p TrustedIssuerDidPolicy

> Verifying from file data/vc/presented/vp-1635188200959.json...  
>  
> Results:  
>  
> SignaturePolicy:		 true  
> JsonSchemaPolicy:		 true  
> TrustedIssuerDidPolicy:		 true  
> Verified:		 true  

_**API**_

Verify:

    curl -X POST "https://auditor.ssikit.walt.id/v1/verify?policyList=SignaturePolicy%2CJsonSchemaPolicy%2CTrustedIssuerDidPolicy" -H  "accept: application/json" -H  "Content-Type: text/plain" -d "{\"@context\":[\"https://www.w3.org/2018/credentials/v1\"],\"credentialSubject\":{\"DNSpublicKey\":\"04:8B:CA:33:B1:A1:3A:69:E6:A2:1E:BE:CB:4E:DF:75:A9:70:8B:AA:51:83:AB:A1:B0:5A:35:20:3D:B4:29:09:AD:67:B4:12:19:3B:6A:B5:7C:12:3D:C4:CA:DD:A5:E0:DA:05:1E:5E:1A:4B:D1:F2:BA:8F:07:4D:C7:B6:AA:23:46\",\"brandName\":\"deltaDAO\",\"commercialRegister\":{\"countryName\":\"Germany\",\"locality\":\"Hamburg\",\"organizationName\":\"Amtsgericht Hamburg (-Mitte)\",\"organizationUnit\":\"Registergericht\",\"postalCode\":\"20355\",\"streetAddress\":\"Caffamacherreihe 20\"},\"corporateEmailAddress\":\"contact@delta-dao.com\",\"ethereumAddress\":{\"id\":\"0x4C84a36fCDb7Bc750294A7f3B5ad5CA8F74C4A52\"},\"id\":\"did:key:z6MkrtMb2FYe6Wqvw8CXUWSQfuieaDBsHY4fhrchmpVw7zYW\",\"individualContactLegal\":\"legal@delta-dao.com\",\"individualContactTechnical\":\"support@delta-dao.com\",\"jurisdiction\":\"Germany\",\"legalForm\":\"Stock Company\",\"legalRegistrationNumber\":\"HRB 170364\",\"legallyBindingAddress\":{\"countryName\":\"Germany\",\"locality\":\"Hamburg\",\"postalCode\":\"22303\",\"streetAddress\":\"Geibelstr. 46B\"},\"legallyBindingName\":\"deltaDAO AG\",\"trustState\":\"trusted\",\"webAddress\":{\"url\":\"https://www.delta-dao.com/\"}},\"id\":\"identity#verifiableID#e378ba65-962f-49b7-8525-b3685f2255c5\",\"issuanceDate\":\"2020-08-24T14:13:44Z\",\"issuer\":\"did:ebsi:zrgUxYjGLSBz2tYW75RUtbt\",\"type\":[\"VerifiableCredential\",\"GaiaxCredential\"],\"proof\":{\"type\":\"EcdsaSecp256k1Signature2019\",\"creator\":\"did:ebsi:zrgUxYjGLSBz2tYW75RUtbt\",\"created\":\"2021-11-09T20:30:01Z\",\"domain\":\"https://api.preprod.ebsi.eu\",\"nonce\":\"5a51cab7-8b8c-4781-8e77-f6fb7cbca30d\",\"jws\":\"eyJiNjQiOmZhbHNlLCJjcml0IjpbImI2NCJdLCJhbGciOiJFUzI1NksifQ..3eDA5x-M8FcqO05LiL1lq6SHNhzlcFJIIQlEQZ3T8weBGGLngs_CBnMSJtFffbOvwtJRemskdaqVLuA4gghULw\"}}"

```json
{
    "overallStatus": true,
    "policyResults": {
        "SignaturePolicy": true,
        "JsonSchemaPolicy": true,
        "TrustedIssuerDidPolicy": true
    }
}
```

### 4.5. Storing a VC in Custodian

```
curl -X PUT "http://localhost:7002/credentials/myvc123" -H  "accept: application/json" -H  "Content-Type: application/json" -d "{\"@context\":[\"https://www.w3.org/2018/credentials/v1\"],\"credentialSchema\":{\"id\":\"https://essif.europa.eu/trusted-schemas-registry/v1/schemas/to_be_obtained_after_registration_of_the_schema\",\"type\":\"JsonSchemaValidator2018\"},\"credentialStatus\":{\"id\":\"https://essif.europa.eu/status/education#higherEducation#51e42fda-cb0a-4333-b6a6-35cb147e1a88\",\"type\":\"CredentialsStatusList2020\"},\"credentialSubject\":{\"awardingOpportunity\":{\"awardingBody\":{\"eidasLegalIdentifier\":\"Unknown\",\"homepage\":\"https://leaston.bcdiploma.com/\",\"id\":\"did:ebsi:2LGKvDMrNUPR6FhSNrXzQQ1h295zr4HwoX9UqvwAsenSKHe9\",\"preferredName\":\"Leaston University\",\"registration\":\"0597065J\"},\"endedAtTime\":\"2020-11-03T00:00:00Z\",\"id\":\"https://leaston.bcdiploma.com/law-economics-management#AwardingOpportunity\",\"identifier\":\"https://certificate-demo.bcdiploma.com/check/87ED2F2270E6C41456E94B86B9D9115B4E35BCCAD200A49B846592C14F79C86BV1Fnbllta0NZTnJkR3lDWlRmTDlSRUJEVFZISmNmYzJhUU5sZUJ5Z2FJSHpWbmZZ\",\"location\":\"FRANCE\",\"startedAtTime\":\"Unknown\"},\"dateOfBirth\":\"1993-04-08\",\"familyName\":\"DOE\",\"givenNames\":\"Jane\",\"gradingScheme\":{\"id\":\"https://leaston.bcdiploma.com/law-economics-management#GradingScheme\",\"title\":\"Lower Second-Class Honours\"},\"id\":\"did:ebsi:22AhtW7XMssv7es4YcQTdV2MCM3c8b1VsiBfi5weHsjcCY9o\",\"identifier\":\"0904008084H\",\"learningAchievement\":{\"additionalNote\":[\"DISTRIBUTION MANAGEMENT\"],\"description\":\"MARKETING AND SALES\",\"id\":\"https://leaston.bcdiploma.com/law-economics-management#LearningAchievment\",\"title\":\"MASTERS LAW, ECONOMICS AND MANAGEMENT\"},\"learningSpecification\":{\"ECTSCreditPoints\":120,\"EQFLevel\":7,\"ISCEDFCode\":[\"7\"],\"NQFLevel\":[\"7\"],\"id\":\"https://leaston.bcdiploma.com/law-economics-management#LearningSpecification\"}},\"id\":\"education#higherEducation#51e42fda-cb0a-4333-b6a6-35cb147e1a88\",\"issuanceDate\":\"2020-11-03T00:00:00Z\",\"issuer\":\"did:ebsi:2LGKvDMrNUPR6FhSNrXzQQ1h295zr4HwoX9UqvwAsenSKHe9\",\"validFrom\":\"2020-11-03T00:00:00Z\",\"type\":[\"VerifiableCredential\",\"VerifiableAttestation\",\"Europass\"]}"
```
