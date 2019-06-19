# test scripts

## prepare access token
```bash
cd ${PROJECT_ROOT}

# variables
USER_TOKEN_PATH=${USER_TOKEN_PATH:-"`pwd`/docker/deploy-scripts/user-token.txt"}
ZMS_ADMIN_PASS=${ZMS_ADMIN_PASS:-replace_me_with_a_strong_passowrd}
DOCKER_NETWORK=${DOCKER_NETWORK:-athenz}
ZMS_HOST=${ZMS_HOST:-athenz-zms-server}
ZMS_PORT=${ZMS_PORT:-4443}
ZTS_HOST=${ZTS_HOST:-athenz-zts-server}
ZTS_PORT=${ZTS_PORT:-8443}
CLI_STATIC_IP=${CLI_STATIC_IP:-172.21.255.254}

# get access token
set -o pipefail
docker run --rm --entrypoint curl \
  --ip "${CLI_STATIC_IP}" \
  --network="${DOCKER_NETWORK}" \
  --name athenz-curl appropriate/curl \
  -k --silent -u "admin:${ZMS_ADMIN_PASS}" \
  "https://${ZMS_HOST}:${ZMS_PORT}/zms/v1/user/_self_/token" \
  | sed 's/^{"token":"//' | sed 's/"}$//' \
  > ${USER_TOKEN_PATH}
rc=$?; if [[ $rc != 0 ]]; then echo "something wrong..."; else printf "\nUser token of admin saved in ${USER_TOKEN_PATH}\n"; fi
```

## acceptance-test for ZTS

- [x] 0. prepare test data
    - [x] 0.1. set up ZMS for testing
    - [x] 0.2. generate n-token
```bash
cd ${PROJECT_ROOT}

# add garm.tester service to ZMS
docker run --rm --net="${DOCKER_NETWORK}" \
  --ip "${CLI_STATIC_IP}" \
  -v "${USER_TOKEN_PATH}:/etc/token/ntoken" \
  -v `pwd`/docker/zms/var/certs/zms_cert.pem:/etc/certs/zms_cert.pem \
  --name athenz-zms-cli athenz-zms-cli \
  -f /etc/token/ntoken -i user.admin \
  -z "https://${ZMS_HOST}:${ZMS_PORT}/zms/v1" -c /etc/certs/zms_cert.pem \
  add-domain garm admin
docker run --rm --net="${DOCKER_NETWORK}" \
  --ip "${CLI_STATIC_IP}" \
  -v "${USER_TOKEN_PATH}:/etc/token/ntoken" \
  -v `pwd`/docker/zms/var/certs/zms_cert.pem:/etc/certs/zms_cert.pem \
  -v `pwd`/docker/acceptance-test/public.pem:/etc/certs/public.pem \
  --name athenz-zms-cli athenz-zms-cli \
  -f /etc/token/ntoken \
  -z "https://${ZMS_HOST}:${ZMS_PORT}/zms/v1" -c /etc/certs/zms_cert.pem \
  -d garm add-service tester tmp /etc/certs/public.pem

# add demo-role to garm domain with member garm.tester to ZMS
docker run --rm --net="${DOCKER_NETWORK}" \
  --ip "${CLI_STATIC_IP}" \
  -v "${USER_TOKEN_PATH}:/etc/token/ntoken" \
  -v `pwd`/docker/zms/var/certs/zms_cert.pem:/etc/certs/zms_cert.pem \
  -v `pwd`/docker/acceptance-test/public.pem:/etc/certs/public.pem \
  --name athenz-zms-cli athenz-zms-cli \
  -f /etc/token/ntoken \
  -z "https://${ZMS_HOST}:${ZMS_PORT}/zms/v1" -c /etc/certs/zms_cert.pem \
  -d garm add-group-role demo-role garm.tester

# add policy to ZMS to test access control
docker run --rm --net="${DOCKER_NETWORK}" \
  --ip "${CLI_STATIC_IP}" \
  -v "${USER_TOKEN_PATH}:/etc/token/ntoken" \
  -v `pwd`/docker/zms/var/certs/zms_cert.pem:/etc/certs/zms_cert.pem \
  -v `pwd`/docker/ui/keys/athenz.ui-server_pub.pem:/etc/certs/athenz.ui-server_pub.pem \
  --name athenz-zms-cli athenz-zms-cli \
  -f /etc/token/ntoken \
  -z "https://${ZMS_HOST}:${ZMS_PORT}/zms/v1" -c /etc/certs/zms_cert.pem \
  -d garm add-policy demo-policy grant get to demo-role on treasure
```
```bash
# generate n-token locally
docker run --rm -v `pwd`/docker/acceptance-test:/etc/acceptance-test \
  --name athenz-cli-util athenz-cli-util \
  ./utils/zms-svctoken/target/linux/zms-svctoken \
  -domain garm -service tester \
  -key-version tmp \
  -private-key /etc/acceptance-test/private.pem > `pwd`/docker/acceptance-test/n-token

### checking:
less ./docker/acceptance-test/n-token
```

- [x] 1. get role token
```bash
# get role token by n-token (may take a moment for ZTS to sync the changes)
docker run --rm --net="${DOCKER_NETWORK}" \
  -v `pwd`/docker/zts/var/certs/zts_cert.pem:/etc/ssl/certs/ca-certificates.crt \
  -v `pwd`/docker/acceptance-test:/etc/acceptance-test \
  --name athenz-cli-util athenz-cli-util \
  ./utils/zts-roletoken/target/linux/zts-roletoken \
  -domain garm -role demo-role \
  -ntoken-file /etc/acceptance-test/n-token \
  -zts "https://${ZTS_HOST}:${ZTS_PORT}/zts/v1"

### expect:
### <role token print in stdout>
```

- [x] 2. get valid service certificate
```bash
# get service certificate for the service
docker run --rm --net="${DOCKER_NETWORK}" \
  -v `pwd`/docker/zts/var/certs/zts_cert.pem:/etc/certs/zts_cert.pem \
  -v `pwd`/docker/acceptance-test:/etc/acceptance-test \
  --name athenz-cli-util athenz-cli-util \
  ./utils/zts-svccert/target/linux/zts-svccert \
  -domain garm -service tester \
  -key-version tmp \
  -private-key /etc/acceptance-test/private.pem \
  -cacert /etc/certs/zts_cert.pem \
  -zts "https://${ZTS_HOST}:${ZTS_PORT}/zts/v1" \
  -dns-domain dns.athenz.cloud \
  -instance dummy-instance \
  -hdr Athenz-Principal-Auth \
  -cert-file /etc/acceptance-test/service.crt \
  -signer-cert-file /etc/acceptance-test/intermediate.crt

### checking:
openssl x509 -text -noout -in ./docker/acceptance-test/service.crt | less
openssl x509 -text -noout -in ./docker/acceptance-test/intermediate.crt | less
openssl verify -CAfile ./docker/zts/var/keys/zts_cert_signer_cert.pem ./docker/acceptance-test/service.crt
### expect:
### ./docker/acceptance-test/service.crt: OK

### checking:
docker run --rm --entrypoint curl \
  --ip "${CLI_STATIC_IP}" \
  --network="${DOCKER_NETWORK}" \
  -v `pwd`/docker/acceptance-test/service.crt:/etc/certs/service.crt \
  -v `pwd`/docker/acceptance-test/private.pem:/etc/certs/private.pem \
  --name athenz-curl appropriate/curl \
  -k --silent --cert /etc/certs/service.crt --key /etc/certs/private.pem \
  "https://${ZTS_HOST}:${ZTS_PORT}/zts/v1/access/get/garm:treasure"
### expect:
### {"granted":true}
```

- [x] 3. get valid role certificate
```bash
# add CA generated in runtime to ZTS truststore
ZTS_CERT_SIGNER_CA_PATH=${ZTS_CERT_SIGNER_CA_PATH:-./docker/acceptance-test/intermediate.crt}
ZTS_SSL_TRUSTSTORE_PATH=${ZTS_SSL_TRUSTSTORE_PATH:-./docker/zts/var/certs/zts_truststore.jks}
ZTS_SSL_TRUSTSTORE_PASS=${ZTS_SSL_TRUSTSTORE_PASS:-athenz}

CERT_ALIAS='dev-zts-cert-signer-ca'
sudo keytool -delete -noprompt -keystore ${ZTS_SSL_TRUSTSTORE_PATH} -storepass "${ZTS_SSL_TRUSTSTORE_PASS}" -storetype JKS -alias "${CERT_ALIAS}"
sudo keytool -importcert -noprompt -keystore ${ZTS_SSL_TRUSTSTORE_PATH} -storepass "${ZTS_SSL_TRUSTSTORE_PASS}" -storetype JKS -alias "${CERT_ALIAS}" -file ${ZTS_CERT_SIGNER_CA_PATH}

# (using make) restart ZTS container
ZTS_CONTAINER=`docker ps -aqf "name=zts-server"`
docker restart "${ZTS_CONTAINER}"

# (using docker-stack) restart ZTS service
rm -f ./docker/logs/zts/server.log
ZTS_SERVICE=`docker stack services -qf "name=athenz_zts-server" athenz`
docker service update --force $ZTS_SERVICE
docker ps -qa -f "label=com.docker.swarm.service.name=athenz_zts-server" -f status=exited | xargs docker rm -f
```
```bash
# get role token by service certificate
docker run --rm --net="${DOCKER_NETWORK}" \
  -v `pwd`/docker/zts/var/certs/zts_cert.pem:/etc/certs/zts_cert.pem \
  -v `pwd`/docker/acceptance-test:/etc/acceptance-test \
  --name athenz-cli-util athenz-cli-util \
  ./utils/zts-rolecert/target/linux/zts-rolecert \
  -domain garm -service tester \
  -svc-key-file /etc/acceptance-test/private.pem \
  -svc-cert-file /etc/acceptance-test/service.crt \
  -cacert /etc/certs/zts_cert.pem \
  -zts "https://${ZTS_HOST}:${ZTS_PORT}/zts/v1" \
  -role-domain garm -role-name demo-role \
  -dns-domain dns.athenz.cloud \
  -role-cert-file /etc/acceptance-test/role.crt

### checking:
openssl x509 -text -noout -in ./docker/acceptance-test/role.crt | less
openssl verify -CAfile ./docker/zts/var/keys/zts_cert_signer_cert.pem ./docker/acceptance-test/role.crt
### expect:
### ./docker/acceptance-test/role.crt: OK

### checking:
docker run --rm --entrypoint curl \
  --ip "${CLI_STATIC_IP}" \
  --network="${DOCKER_NETWORK}" \
  -v `pwd`/docker/acceptance-test/role.crt:/etc/certs/role.crt \
  -v `pwd`/docker/acceptance-test/private.pem:/etc/certs/private.pem \
  --name athenz-curl appropriate/curl \
  -k --silent --cert /etc/certs/role.crt --key /etc/certs/private.pem \
  "https://${ZTS_HOST}:${ZTS_PORT}/zts/v1/access/get/garm:treasure"
### expect:
### {"granted":true}
```

- [x] 4. get policy
```bash
# get policy - ZPU
rm -f ./docker/acceptance-test/zpu.log
mkdir -p ./docker/acceptance-test/zpu/metrics
docker run --rm --net="${DOCKER_NETWORK}" \
  -v `pwd`/docker/zts/var/certs/zts_cert.pem:/etc/certs/zts_cert.pem \
  -v `pwd`/docker/acceptance-test:/etc/acceptance-test \
  -v `pwd`/docker/acceptance-test/zpu:/home/athenz/tmp/zpe \
  -v `pwd`/docker/zts/conf/athenz.conf:/tmp/athenz.conf \
  --name athenz-cli-util athenz-cli-util \
  ./utils/zpe-updater/target/linux/zpu \
  -athenzConf /tmp/athenz.conf -zpuConf /etc/acceptance-test/zpu.conf \
  -logFile /etc/acceptance-test/zpu.log

### checking:
less ./docker/acceptance-test/zpu.log
# cat ./docker/acceptance-test/zpu/garm.pol
```
