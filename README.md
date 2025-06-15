# kubecon2025-demo

This is README for run the demo of the standard token exchange and fine-grained admin permissions used during KubeCon + CloudNativeCon Japan 2025.

1) Run Keycloak 26.2.4 or newer on your laptop with the command like:

```
./kc.sh start-dev
```

So for this demo, assume that Keycloak is running on http://localhost:8080 .

2) Open http://localhost:8080 and create the initial admin user. Then login to the admin console.

3) Go to the screen for importing realms in the admin console and import [token-exchange-demo.json] file from this directory. See [Keycloak documentation](https://www.keycloak.org/server/importExport#_importing_and_exporting_by_using_the_admin_console) for more details about realm import. This initial realm is to create initial structure of the clients. Similar structure (with same names) is also described in the official [https://www.keycloak.org/securing-apps/token-exchange#_standard-token-exchange-flow](Keycloak token exchange documentation).

4) Now login from the command line. For the demo purpose, we just use OAuth2 Resource owner password credentials grant for the login from terminal. Assuming you have linux terminal with the utilities like `curl` and `jwtp`, you can run these commands for the initial user authentication and obtaining of the initial subject token

```
KC_REALM=test
KC_USER=john
KC_PASSWORD=john
KC_INITIAL_CLIENT=initial-client
KC_INITIAL_CLIENT_SECRET=rS6yAuHYyMmRdeNxVL2ECuaIZXyzQMEZ

SUBJECT_TOKEN=`curl -L -X POST "http://localhost:8080/realms/$KC_REALM/protocol/openid-connect/token" \
-u "$KC_INITIAL_CLIENT:$KC_INITIAL_CLIENT_SECRET" \
-H 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode "client_id=$KC_INITIAL_CLIENT" \
--data-urlencode "client_secret=$KC_INITIAL_CLIENT_SECRET" \
--data-urlencode 'grant_type=password' \
--data-urlencode "username=$KC_USER" \
--data-urlencode "password=$KC_PASSWORD" | awk -F  "\"" '{print $4}'`

echo subject_token=$SUBJECT_TOKEN
jwtp $SUBJECT_TOKEN
```

You can notice token was issued to the initial-client (claim `azp`) and only audience is `requester-client` (Claim `aud`). See the [https://www.keycloak.org/docs/latest/server_admin/index.html#audience-support](Keycloak documentation) for the details on how the audiences can be set in the Keycloak token.

5) Token exchange request - without scope and audience parameters. 

```
KC_REQUESTER_CLIENT=requester-client
KC_REQUESTER_CLIENT_SECRET=tEgsPgnsk1wLbmZMsNSUaBFuTG0BIl7q

EXCHANGED_TOKEN=`curl -L -X POST "http://localhost:8080/realms/$KC_REALM/protocol/openid-connect/token" \
-u "$KC_REQUESTER_CLIENT:$KC_REQUESTER_CLIENT_SECRET" \
-H 'Content-Type: application/x-www-form-urlencoded' \
-H 'Accept: application/json' \
--data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:token-exchange" \
--data-urlencode "subject_token=$SUBJECT_TOKEN" \
--data-urlencode "subject_token_type=urn:ietf:params:oauth:token-type:access_token" \
--data-urlencode "requested_token_type=urn:ietf:params:oauth:token-type:access_token" | awk -F  "\"" '{print $4}'`

echo exchanged_token=$EXCHANGED_TOKEN
jwtp $EXCHANGED_TOKEN
```

You can notice client is issued to the `requester-client` and only audience is `target-client1`. That audience is available because of the default client scope `default-scope1`, which is the default client scope for the `requester-client` and you can see it in the `scope` of the token. See [https://www.keycloak.org/docs/latest/server_admin/index.html#_client_scopes_linking](this page) for the details about the client scopes and their linking to clients.



6) Repeat the same CURL request, but now also add scope parameter to the request by adding this additional argument to existing parameters of the `curl` command from above:
```
--data-urlencode "scope=optional-scope2"
```

You can see that now in the token there is also `optional-client-scope2` included and because of that, there is also audience `target-client2` included.


7) Add also audience parameter with the value `target-client2` . It can be done by attaching this to the parameters from the previous step:
```
--data-urlencode "audience=target-client2"
```

Parameter audience is used to filter audiences, so all other audiences besides the requested one are omitted. So in this case only `target-client2` would be used as an audience and also the `optional-scope2` would be only client scope used.

8) Let's try to use `target-client3` as an audience:
```
--data-urlencode "audience=target-client3"
```

The request will fail because audience `target-client3` is not available as an audience. You can also see Warning in the server log about this.


