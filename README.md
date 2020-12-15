# Example of KrakenD and Keycloak Integration

## Description

basato su https://github.com/xyder/example-krakend-keycloak

integrazione KrakenD and Keycloak

### Deployment

1. entrare dentro keycloak http://localhost:8403   (admin:admin)
2. andare su client e creare un nuovo client (esempio myclient) (KEYCLOAK-CLIENT-ID)
3. su Access Type selezionare: "confidential"
4. su Valid Redirect URIs inserire: *
5. su tab "Credential" copiare il secret (KEYCLOAK-SECRET-ID)
6. andare su users e creare un nuovo user: create user
7. sul tab Credential settare password e confirm password  temporary: off save 
8. modificare data/krakend/krakend.json e modificare la riga:
"jwk-url": "http://IP_PUBLIC_KEYCLOAK:8403/auth/realms/master/protocol/openid-connect/certs",
con IP_PUBLIC_KEYCLOAK raggiungibile da krakend oppure pubblico
9. Deploy docker containers with `docker-compose up -d`


```bash
# api per recuperare il token da keycloak:
curl -d 'client_secret=KEYCLOAK-SECRET-ID' -d 'client_id=KEYCLOAK-CLIENT-ID' -d 'username=valerio' -d 'password=valerio' -d 'grant_type=password' 'http://localhost:8403/auth/realms/master/protocol/openid-connect/token' 

export TOKEN=eyJhbGciOiJSUzI...(truncated)

#api per chiamare krakend usarlo come gateway e validare il token di keycloak:
curl -H "Authorization: Bearer $TOKEN" http://localhost:8402/mock/parents/1 -i


```

### config
```json
{
    "version": 2,
    "extra_config": {
        "github_com/devopsfaith/krakend-gologging": {
            "level": "DEBUG",
            "prefix": "[KRAKEND]",
            "syslog": false,
            "stdout": true,
            "format": "default"
        }
    },
    "timeout": "3000ms",
    "cache_ttl": "300s",
    "output_encoding": "json",
    "name": "Test Service",
    "endpoints": [
        {
            "endpoint": "/mock/parents/{id}",     -------->  questo è l'end-point richiamabile su krakend
            "method": "GET",
           "headers_to_pass": [
                "Authorization"
            ],
            "extra_config": {
                "github.com/devopsfaith/krakend-jose/validator": {
                    "alg": "RS256",
					"jwk-url": "http://IP_PUBLIC_KEYCLOAK:8403/auth/realms/master/protocol/openid-connect/certs",   ----> ip pubblico raggiungibile da krakend
                    "issuer": "http://localhost:8403/auth/realms/master",    ----> lasciare localhost (è proprio iss definito nel token)
                    "disable_jwk_security": true
                }
            },
            "output_encoding": "json",
            "concurrent_calls": 1,
            "backend": [
                {
                    "url_pattern": "/todos/{id}",   ----> path relativo da richiamare rispetto a https://jsonplaceholder.typicode.com
                    "encoding": "json",
                    "sd": "static",
                     "extra_config": {},
                    "host": [
                        "https://jsonplaceholder.typicode.com"     ----->  base url dove girare la richiesta
                    ],
                    "disable_host_sanitize": false,
                    "blacklist": [
                        "super_secret_field"
                    ]
                }
            ]
        }
    ]
}
```


### role mapping tricks
1. fare in modo che i ruoli nel jwt non siano in realm_access.roles ma direttamente in roles(krakend non gestisce i dati annidati nel JWT) (https://discuss.istio.io/t/nested-jwt-claims-validation/3583/3)
nel clients-->client id--> mapper creare un mappper: 
-name:role-mapper  
-Mapper Type "User Realm Role"  
-Token Claim Name: roles  
-Claim JSON Type:String  
-Add to ID token:off 
-Add to access token:on 
-Add to userinfo:off
2. aggiungere nell'extra_config roles_key(roles) e in roles i ruoli che devono esistere affinche avvenga il routing:
```json
 "extra_config": {
                "github.com/devopsfaith/krakend-jose/validator": {
                    "alg": "RS256",
					 "roles_key": "roles",
                     "roles": ["myrole"],
					"jwk-url": "http://192.168.0.8:8403/auth/realms/master/protocol/openid-connect/certs",
                    "issuer": "http://localhost:8403/auth/realms/master",
                    "disable_jwk_security": true
                }
				
            },
```			
