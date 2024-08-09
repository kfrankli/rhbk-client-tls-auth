# Configurinng X.509 user1 Certificate Based User Authentication in Red Hat Build of Keycloak with a Customer Trust Heirarchy [RHBK]

> [!NOTE]
> If you found this repo helpful or you found a bug, please feel free to submit a PR, drop me an email kfrankli@redhat.com, or even just star the repo. I welcome any and all feedback. 

## Overview

This repository outlines the steps needed to configure X.509 certificate based user authentication. It includes not only the steps needed to configure RHBK (Red Hat build of Keycloak), but also how to create a simple certificate trust heirarchy to test this.

Note this is made available for demonstration purposes and is not representative of a production ready configuration. Further steps would be required.

## Instructions

### Creating our X509 Heirarchy

1.  Make directories we will need:

    ```console
    mkdir -p rhbk-testing/CA
    ```

2.  Change directories:

    ```console
    cd rhbk-testing/CA
    ```

3.  We're going to have to create a X509 certificate trust heirarchy in order to properly test this out. We'll first create a root-CA and 2 end-entity certificates for the user1 and server that we will use to test. Since we're creating a new root-CA, we will also have to configure the trust heirarchy.

    ![Trust Heirarchy Diagram](./images/heirarchy.png)

4.  Generate the private key and pem encoded certificate for the root-CA:

    ```console
    openssl ecparam -out CA.consulting.redhat.com.key -name secp384r1 -genkey

    openssl req -x509 -new -key CA.consulting.redhat.com.key -out CA.consulting.redhat.com.crt -outform pem -sha384 -subj "/C=US/ST=VA/L=TYSONS/O=RED HAT/OU=CONSULTING/CN=CA.consulting.redhat.com"
    ```

5.  Now that we have our CA, we will generate private keys for the server (`rhbksrvr`) and client (`user1`)

    ```console
    openssl ecparam -out rhbksrvr.consulting.redhat.com.key -name secp384r1 -genkey

    openssl ecparam -out user1.consulting.redhat.com.key -name secp384r1 -genkey
    ```

6.  Create a Certificate Signing Request (CSR) for the newly created server (`rhbksrvr`) and client (`user1`) keys:

    ```console
    openssl req -new -nodes -key rhbksrvr.consulting.redhat.com.key -outform pem -out rhbksrvr.consulting.redhat.com.csr -sha384 -subj "/C=US/ST=VA/L=TYSONS/O=RED HAT/OU=CONSULTING/CN=rhbksrvr.consulting.redhat.com"

    openssl req -new -nodes -key user1.consulting.redhat.com.key -outform pem -out user1.consulting.redhat.com.csr -sha384 -subj "/C=US/ST=VA/L=TYSONS/O=RED HAT/OU=CONSULTING/CN=user1.consulting.redhat.com"
    ```

7.  Use the root-CA you created in step 4 to sign the server and user1 CSRs:

    ```console
    openssl x509 -req -in rhbksrvr.consulting.redhat.com.csr -CA CA.consulting.redhat.com.crt -CAkey CA.consulting.redhat.com.key -CAcreateserial -out rhbksrvr.consulting.redhat.com.crt -days 2048 -sha384

    openssl x509 -req -in user1.consulting.redhat.com.csr -CA CA.consulting.redhat.com.crt -CAkey CA.consulting.redhat.com.key -CAcreateserial -out user1.consulting.redhat.com.crt -days 2048 -sha384
    ```

8.  Convert the pem format certicate to PKCS12 format:

    ```console
    openssl pkcs12 -export -out rhbksrvr.consulting.redhat.com.p12 -inkey rhbksrvr.consulting.redhat.com.key -in rhbksrvr.consulting.redhat.com.crt -certfile CA.consulting.redhat.com.crt -password pass:JBossRocks#123 -name "rhbksrvr.consulting.redhat.com"

    openssl pkcs12 -export -out user1.consulting.redhat.com.p12 -inkey user1.consulting.redhat.com.key -in user1.consulting.redhat.com.crt -certfile CA.consulting.redhat.com.crt -password pass:JBossRocks#123 -name "user1.consulting.redhat.com"
    ```

9.  Now convert the PKCS12 format files to Java Keystore (JKS) format:

    ```console
    keytool -importkeystore -destkeystore rhbksrvr.consulting.redhat.com.jks -srckeystore rhbksrvr.consulting.redhat.com.p12 -srcstoretype PKCS12 -srcalias "rhbksrvr.consulting.redhat.com" -destalias "rhbksrvr.consulting.redhat.com" -srcstorepass JBossRocks#123 -deststorepass JBossRocks#123

    keytool -importkeystore -destkeystore user1.consulting.redhat.com.jks -srckeystore user1.consulting.redhat.com.p12 -srcstoretype PKCS12 -srcalias "user1.consulting.redhat.com" -destalias "user1.consulting.redhat.com" -srcstorepass JBossRocks#123 -deststorepass JBossRocks#123
    ```

10. Let's add our root CA to a trusted CAs file:

    ```console
    cat CA.consulting.redhat.com.crt >> CA.crt
    ```

11. Convert that trusted CAs fiile to a java keystore truststore:

    ```console
    keytool -import -alias "CA.consulting.redhat.com" -file CA.consulting.redhat.com.crt -keystore trusts.jks -storepass JBossRocks#123 -noprompt
    ```

### Installing and configuring RHBK

1.  Download `Red Hat build of Keycloak 24.0.6 Server`: [Red Hat build of Keycloak Download](https://access.redhat.com/jbossnetwork/restricted/listSoftware.html?product=rhbk&downloadType=distributions)

    ![Software Downloads](./images/download.png)

2.  Switch up a directory so that you're in `./rhbk-testing`

    ```console
    cd ..
    ```

3.  Copy the downloaded RHBK to the `./rhbk-testing` directory

    ```console
    cp ~/Downloads/rhbk-24.0.6.zip .
    ```

4.  Unzip RHBK:

    ```console
    unzip rhbk-24.0.6.zip
    ```

5.  Change directories into the new rhbk directory:

    ```console
    cd ./rhbk-24.0.6
    ```

6.  Copy the previously created truststore (`trusts.jks`) and the server's PEM encoded certificate/key pair (`rhbksrvr.consulting.redhat.com.crt` & `rhbksrvr.consulting.redhat.com.key`) into the current directory:

    ```console
    cp ../CA/trusts.jks .
    cp ../CA/rhbksrvr.consulting.redhat.com.crt .
    cp ../CA/rhbksrvr.consulting.redhat.com.key .
    ```

7.  Add the PKCS12 certificate/key pair file `rhbk-testing/CA/user1.consulting.redhat.com.p12` to your browser with password `JBossRocks#123`

    ![Chrome Certificate Viewer](./images/cert-viewer.png)

8.  We'll now start rhbk. We will need to pass in the following arguments though:

    | Argument                          | Value                                     | Purpose |
    | --------------------------------- | ----------------------------------------- |---------|
    | `--hostname`                      | `0.0.0.0`                                 | Allowing us to explicitly set the hostname RHBK will listen on |
    | `--https-certificate-file`        | `./rhbksrvr.consulting.redhat.com.crt`    | The location of the public key certificate that RHBK will present to clients connecting |
    | `--https-certificate-key-file`    | `./rhbksrvr.consulting.redhat.com.key `   | The location of the private key that RHBK will present to clients connecting |
    | `--https-trust-store-file`        | `./trusts.jks `                           | The location of the truststore to be used for validating connecting clientts in the case of mutual TLS|
    | `--https-trust-store-password`    | `JBossRocks#123 `                         | The password for the https-trust-store-file |
    | `--https-trust-store-type`        | `jks`                                     | The format of https-trust-store-file |
    | `--https-client-auth`             | `request`                                 | By setting request, RHBK will request client certificates as part of a mTLS connection. It will still succeed without a client certificate when set to `request`. To mandate client certificates set this to `required` |

    ```console
    ./bin/kc.sh start --https-certificate-file=./rhbksrvr.consulting.redhat.com.crt --https-certificate-key-file=./rhbksrvr.consulting.redhat.com.key --https-trust-store-file=./trusts.jks --https-trust-store-password=JBossRocks#123 --https-trust-store-type=jks --hostname=0.0.0.0 --hostname-debug=true --https-client-auth=request --verbose
    ```

9.  Once RHBK is up and running, open https://0.0.0.0:8443 in the browser of your choice.

Open brower to 
Confirm the invalid cert is in fact:
    Common Name (CN)	rhbksrvr.consulting.redhat.com
    Organization (O)	RED HAT
    Organizational Unit (OU)	CONSULTING
Create an admin user
    admin
    password123



Select dropdown top left to `Create realm`

Name realm `test`, then `Create`

Go to Authentication

Duplicate browser, name x509-browser-flow

Add a step `X509/Validate Username Form`. Then shift it to below `Identity Provider Redirector`

Modify `X509/Validate Username Form`. Set `Alias` to `test`. Alter the `A regular expression to extract user idenity` field to `CN=(.*?)(?:,|$)`

On `Action` menu dropdown, select `Bind flow`. Select `Browser flow` and click `Save`

Go to Users

Hit `Add user`, add `user1.consulting.redhat.com`. Click `Create`

`Credentails` tab > `Set password`.

Enter a password and confirm. Uncheck `Temporary` and then `Save`, then `Save password`.

https://0.0.0.0:8443/realms/test/account/#/


reset; openssl s_user1 -connect 0.0.0.0:8443

This shows consulting isn't in my requests./.. ugh

./bin/kc.sh start --https-certificate-file=./rhbksrvr.consulting.redhat.com.crt --https-certificate-key-file=./rhbksrvr.consulting.redhat.com.key --hostname=sso.server.lab --https-trust-store-file=./trusts.jks --https-trust-store-password=JBossRocks#123 --https-trust-store-type=jks --hostname=0.0.0.0 --hostname-debug=true --https-user1-auth=request


===============================

cp ~/Downloads/rhbk-24.0.5.zip .

unzip rhbk-24.0.5.zip

cd ./rhbk-24.0.5

cp ../CA/trusts.jks .
cp ../CA/rhbksrvr.consulting.redhat.com.crt .
cp ../CA/rhbksrvr.consulting.redhat.com.key .


./bin/kc.sh start-dev --https-certificate-file=./rhbksrvr.consulting.redhat.com.crt --https-certificate-key-file=./rhbksrvr.consulting.redhat.com.key --hostname=sso.server.lab --config-keystore=./trusts.jks --config-keystore-password JBossRocks#123 --hostname=0.0.0.0 --hostname-debug=true


start-dev

./bin/kc.sh start --https-certificate-file=./rhbksrvr.consulting.redhat.com.crt --https-certificate-key-file=./rhbksrvr.consulting.redhat.com.key --hostname=sso.server.lab --config-keystore=./trusts.jks --config-keystore-password JBossRocks#123 --hostname=0.0.0.0 --hostname-debug=true
```

Open brower to https://0.0.0.0:8443
Confirm the invalid cert is in fact:
    Common Name (CN)	rhbksrvr.consulting.redhat.com
    Organization (O)	RED HAT
    Organizational Unit (OU)	CONSULTING
Create an admin user
    admin
    password123

