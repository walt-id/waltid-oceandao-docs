# Instructions for the Walt.ID SSI-Kit custom buid for deltaDAO

For easy reproducibility, we have published a custom docker container (at docker hub with the tag "1.0-SNAPSHOT-custom-deltadao") so that you can follow these examples without having to setup a SSI-Kit environment. Our dockerfiles are also podman compatible, if you'd rather like to use podman instead of docker.

## Setting up Walt.ID SSI-Kit

### 1. Initiating directory with the Walt.ID SSI-Kit
In the after-next step, we will use docker to create a container mount, which will be owned by root:root. To not have random root-owned directories laying around, we recommend first creating a new directory.

    mkdir issuer-service && cd issuer-service

Pull the container:

    docker pull docker.io/waltid/ssikit:1.0-SNAPSHOT-custom-deltadao

### 2. Create an alias

We will continue to create an alias, otherwise the commands will be incredibly long. Of course this isn't actually mandatory, you can easily replace all instances of the alias-command "ssikit" with the content between the quotes in the next line.

    alias ssikit="docker container run -p 7000-7004:7000-7004 -itv $(pwd)/data:/app/data docker.io/waltid/ssikit:1.0-SNAPSHOT-custom-deltadao"
    ssikit -h

Now all sensitive data, like cryptographic key, DIDs and credentials will be stored with root-rights in the "data" directory.

## 2. Setting up an Issuer on the EBSI ledger

### 1. Key Creation
We start out by generating a Secp256k1 key.

    ssikit key gen -a Secp256k1

> Generating Secp256k1 key pair...  
> Key "efc581186a0945d8af75cdb8e1b16033" generated.

### 2. DID EBSI creation
We contine to create a did:ebsi with the key which we have just generated.

    ssikit did create -m ebsi -k efc581186a0945d8af75cdb8e1b16033
    # obviously we'll use the key here that we've just generated

> DID created: did:ebsi:zyc8qqkQifbyqZF7GGHW8zS  
> DID document (below, JSON): ...

### 3. Registering DID EBSI on the ledger

To use the EBSI API, we first have to get our EBSI bearer token.

On https://app.preprod.ebsi.eu/users-onboarding click "Onboard with Captcha" if you don't have an EU account. Choose "Desktop Wallet" to have the session token displayed (Mobile Wallet will show the token encoded in a QR code).

Save your bearer-token somewhere the container is able to access, e.g. `data/bearer-token.txt`. Depending on your system, you'll have to write there as `root` user.

    echo "eyJhbGc_YOUR_TOKEN_HERE..." | sudo tee data/bearer-token.txt

#### 3.1. Onboarding flow:
Enter the did:ebsi that was generated above, 

    ssikit essif onboard --did did:ebsi:zyc8qqkQifbyqZF7GGHW8zS data/bearer-token.txt

> ESSIF onboarding of did:ebsi:zyc8qqkQifbyqZF7GGHW8zS...  
> ESSIF onboarding for DID did:ebsi:zyc8qqkQifbyqZF7GGHW8zS was performed successfully.

#### 3.2 Authentication API flow:

    ssikit essif auth-api --did did:ebsi:zyc8qqkQifbyqZF7GGHW8zS

> Running EBSI Authentication API flow...  
> EBSI Authorization flow was performed successfully.

#### 3.3 Writing to ledger (signing of ETH transaction):

    ssikit essif did register --did did:ebsi:zrsRwW1wFHcrBsFazkq3yUf

> Registering DID did:ebsi:zyc8qqkQifbyqZF7GGHW8zS on the EBSI ledger...  
> DID registration was performed successfully. Call command: 'did resolve --did did:ebsi:zyc8qqkQifbyqZF7GGHW8zS' in order to retrieve the DID document from the EBSI ledger.

#### 3.4 Try out DID resolving via CLI:

    ssikit did resolve --did did:ebsi:zrsRwW1wFHcrBsFazkq3yUf

> Resolving DID "did:ebsi:zyc8qqkQifbyqZF7GGHW8zS"...  
> Result: ...

Alternatively, the DID can also be resolved directly using the ESSIF API:

Their swagger interface: https://api.preprod.ebsi.eu/docs/?urls.primaryName=DID%20Registry%20API#/DID%20Registry/get-did-registry-v2-identifier

## 4. Issuing a custom credential for the deltaDAO Datamarketplace

We'll issue a custom credential based on the schema deltaDAO provided, the issuer being a subject with a did:ebsi key (the one we've just created and registered to EBSI above), and the holder having another key format, we'll use a normal did:key for this demo. You may also mix in other DIDs (e.g. for the verifier), like did:web.

### 4.1. Creating a DID for the recipient (holder) of the credential 
Business as usual:

    ssikit did create -m key

### 4.2. Issuing W3C Verifiable Credential based on the deltaDAO credential schema (with the CLI)
To try this example with the REST API, see "Via REST (Swagger)".

We'll use the interactive CLI data provider for this template.

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

### 4.3. Creating a W3C Verifiable Presentation

Using the credential we've just generated, we will generate a Verifiable Presentation (VP) as a holder (with the did:key).

    ssikit vc present --holder-did did:key:z6MkerDxjrmLUKovtbiAYRQYLKHb6cguSRhucaz7nXN9ZtLw data/vc.json

> Creating verifiable presentation from files...  
> Saved verifiable presentation to: "data/vc/presented/vp-1635188200959.json"  

### 4.4. Verifying the VP

#### Via CLI
    ssikit vc verify data/vc/presented/vp-1635188200959.json -p SignaturePolicy -p JsonSchemaPolicy -p TrustedIssuerDidPolicy

> Verifying from file data/vc/presented/vp-1635188200959.json...  
>  
> Results:  
>  
> SignaturePolicy:		 true  
> JsonSchemaPolicy:		 true  
> TrustedIssuerDidPolicy:		 true  
> Verified:		 true  

#### 5. Via REST (Swagger)

    ssikit serve
> Walt.ID SSI-Kit 1.0-SNAPSHOT (deltaDAO custom release) (running on Java 16.0.1+9-24)  
>   
>  walt.id Core API:      http://127.0.0.1:7000  
>  walt.id Signatory API: http://127.0.0.1:7001  
>  walt.id Custodian API: http://127.0.0.1:7002  
>  walt.id Auditor API:   http://127.0.0.1:7003  
>  walt.id ESSIF API:     http://127.0.0.1:7004  
