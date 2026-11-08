# Setting up EJBCA with Docker Compose

This guide will walk you through the process of setting up EJBCA using Docker Compose. EJBCA is a free and open-source public key infrastructure (PKI) and certificate authority (CA) software package originally created by PrimeKey Solutions AB and now maintained by Keyfactor.

## Prerequisites

Before you begin, you'll need to have the following installed on your system:

- Docker
- Docker Compose

Podman and Podman Compose should work as alternatives.

## Installation

To run EJBCA using Docker Compose, follow these steps:

1. Open ports to allow access to the services.<br>
- Redhat, CentOS, Rocky Linux
    ```
    sudo firewall-cmd --permanent --zone=public --add-port={80,443}/tcp
    sudo firewall-cmd --reload
    ```
- Ubuntu
    ```
    sudo ufw allow proto tcp from any to any port 80,443
    ```

2. Edit the `.env` environment file:

    `.env` holds the passwords, ports, versions, and TLS mode. Change the passwords before your first start.

3. Start the EJBCA application:

    ```bash
    docker compose up -d
    ```

   This command will start the EJBCA application using the 'docker-compose.yml' file.  The first start takes a few minutes.

4. Access the EJBCA web interface:

   If logging in with the default TLS_SETUP_ENABLED=true, then EJBCA creates a ManagementCA and a one-time enrollment for a superadmin user.

   - Find the enrollment URL and password with `docker compose logs ejbca | grep -i -A5 superadmin`.
   - Open https://<EJBCA_HOSTNAME>/ejbca/ra/enrollwithusername.xhtml?username=superadmin.
   - Enter the password from the logs, generate the PKCS#12 client certificate, and import it into your browser.
     *Note:* The default selected algorithm may not be supported.  Be sure to choose your desired algorithm, like 'RSA 4096' or what ever.
   - Go to https://<EJBCA_HOSTNAME>/ejbca/adminweb/.

   If TLS_SETUP_ENABLED=simple, then the admin UI io open to anyone over HTTPS without a client certificate.  This is only useful for 'testing' scenarios, so use it only on a throwaway local instance.

## Recommendation for production environments:

1. Pin EJBCA_VERSION to a specific tag instead of latest.
2. Back up the ejbca-db-data volume. It holds all CA keys (when using soft tokens), certificates, and configuration.
3. Consider putting EJBCA behind a reverse proxy, or using an HSM for CA keys.

## Testing the installation

A good test covers the full certificate lifecycle: the server is healthy, it can issue a certificate, and it can revoke one and report that revocation through CRL and OCSP. The steps below use the EJBCA command-line tool inside the container, plus openssl and curl on the host. Every EJBCA CLI command accepts --help if a flag differs in your version.

To save typing, set an alias on the host first:

~~~bash
alias ejbca='docker exec -it ejbca /opt/keyfactor/bin/ejbca.sh'
~~~

1. Check that the containers and the server are healthy
    ~~~bash
    docker compose ps                          # both should be "running", db "healthy"
    docker compose logs ejbca | grep -iE "error|started"
    ejbca ca listcas                           # should list ManagementCA (with TLS_SETUP_ENABLED=true)
    ~~~
    EJBCA also has a built-in health endpoint that returns ALLOK when the database, CAs, and memory are fine:

    ~~~bash
    curl http://localhost/ejbca/publicweb/healthcheck/ejbcahealth
    ~~~
    By default this endpoint only answers requests from localhost. If you get an authorization error from the host, that’s expected. Either run it from inside the container (if the image has curl) or add your IP under healthcheck.authorizedips.
2. Confirm TLS and admin access
    ~~~bash
    openssl s_client -connect localhost:443 -showcerts </dev/null | openssl x509 -noout -subject -issuer
    ~~~
    The server certificate should be issued by ManagementCA. Then log in to https://localhost/ejbca/adminweb/ with your superadmin certificate. Reaching the dashboard proves client-certificate authentication works.
3. Create a throwaway test CA
    ~~~bash
    ejbca ca init --caname TestCA --dn "CN=Test CA,O=Example" \
    --tokenType soft --tokenPass null --keytype RSA --keyspec 3072 \
    -v 3650 --policy null -s SHA256WithRSA

    ejbca ca getcacert --caname TestCA -f /tmp/testca.pem
    docker cp ejbca:/tmp/testca.pem .
    ~~~
4. Issue a certificate from a CSR

    Create a key and CSR on the host, then register an end entity that accepts a user-generated key:

    ~~~bash
    openssl req -new -newkey rsa:2048 -nodes -keyout test.key -out test.csr -subj "/CN=testuser"
    docker cp test.csr ejbca:/tmp/test.csr

    ejbca ra addendentity --username testuser --dn "CN=testuser" --caname TestCA \
    --type 1 --token USERGENERATED --password foo123 \
    --certprofile ENDUSER --eeprofile EMPTY

    ejbca createcert --username testuser --password foo123 -c /tmp/test.csr -f /tmp/test.pem
    docker cp ejbca:/tmp/test.pem .
    ~~~

    Verify that the certificate chains to your CA and matches your key:

    ~~~bash
    openssl verify -CAfile testca.pem test.pem        # expect "test.pem: OK"
    openssl x509 -in test.pem -noout -text | head -20
    diff <(openssl x509 -in test.pem -noout -pubkey) <(openssl pkey -in test.key -pubout)
    ~~~
5. Check OCSP before and after revocation
    ~~~bash
    openssl ocsp -issuer testca.pem -cert test.pem \
    -url http://localhost/ejbca/publicweb/status/ocsp -resp_text | grep -i "cert status"  # expect: good

    ejbca ra revokeendentity --username testuser -r 1     # reason 1 = keyCompromise

    openssl ocsp -issuer testca.pem -cert test.pem \
    -url http://localhost/ejbca/publicweb/status/ocsp -resp_text | grep -i "cert status"  # expect: revoked
    ~~~
6. Check the CRL
    ~~~bash
    ejbca ca createcrl --caname TestCA
    curl -o test.crl "http://localhost/ejbca/publicweb/webdist/certdist?cmd=crl&issuer=CN=Test%20CA,O=Example"
    openssl crl -inform DER -in test.crl -noout -text | grep -A1 "Serial Number"
    openssl x509 -in test.pem -noout -serial          # this serial should appear in the CRL
    ~~~
7. Test persistence
    Restart EJBCA
    ~~~bash
    docker compose down && docker compose up -d
    ejbca ca listcas
    ~~~
    After the restart, TestCA should still be listed, and ejbca ra findendentity --username testuser should still show the revoked user. This confirms the database volume is working. Don’t use docker compose down -v, because the -v flag deletes the volume.

8. Optional: test the web workflows

   To check the UI path that real users will take, open the RA web at https://localhost/ejbca/ra/ and use Make New Request to enroll a certificate through the browser. If you plan to use protocols like ACME, SCEP, EST, or the REST API, enable each one under System Configuration → Protocol Configuration in the admin web, then test it with a real client. For example, certbot works for ACME and curl for REST.

Cleanup

When you’re done, revoke and delete the test entities. Then remove TestCA from CA Functions in the admin web, or keep it as a sandbox. A clean pass through steps 1–7 means EJBCA can issue, validate, revoke, and publish status, and keeps its data across restarts.
