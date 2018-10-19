# okta_authentication_mfa_flow

Sample docs and Postman Collection for using Okta's Authentication API and Factors API.

## Use Case

This document illustrates the basic API calls needed to register a user, enroll them in MFA factors, challenge the user for an MFA factor, and check for an active Okta session. The API calls and attached Postman Collection are mapped to Okta's Transaction State model to help better understand the various states of a user account in Okta, and how MFA policies are used to determine when a user must enroll in a factor.

## Okta Authentication and MFA flow

The documentation below and the included Postman collection invoke Okta's REST APIs directly. Okta also provides a JavaScript library that wraps some of these calls to make deploying a custom sign-in page even easier. Okta's Auth JavaScript SDK can be found on Github: [okta-auth-js](https://github.com/okta/okta-auth-js). 

##Transaction State

![Transaction State](https://developer.okta.com/assets/auth-state-model-48fff4b8914b83daa082cd43df970a643876454227f5eedc5729cbb55d5d4d64.png)

###Authentication API (Primary Authentication)

https://developer.okta.com/docs/api/resources/authn#primary-authentication
__Postman Request: _Primary Authentication___
```bash
curl -X POST \
  https://oktalane.okta.com/api/v1/authn \
  -H 'Accept: application/json' \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: b9e5e0cd-a8e1-44c2-9cdf-25f0b5f5daf2' \
  -d '{
  "username": "jeff.beck",
  "password": "xaxaxaxaxa",
  "options": {
    "multiOptionalFactorEnroll": true,
    "warnBeforePasswordExpired": false
  }  
}'
```

####status: SUCCESS

If no policy rule requires MFA, the response status will be __SUCCESS__ and include a sessionToken that can be exchanged for a sessionCookie (see below).

```json
{
    "expiresAt": "2018-10-03T18:32:22.000Z",
    "status": "SUCCESS",
    "sessionToken": "20111BRxe-iyWLZ0TnEN7zwaNTZ9LaY4tf3MwSYKju8kZMx-f-TTC-d",
    "_embedded": {
        "user": {
            "id": "00u2e9pzr0aCdtXuI1t3",
            "passwordChanged": "2017-05-24T19:52:29.000Z",
            "profile": {
                "login": "jeff.beck@acme.com",
                "firstName": "Jeff",
                "lastName": "Beck",
                "locale": "en",
                "timeZone": "America/Los_Angeles"
            }
        }
    }
}
```

To exhange the sessionToken for a Okta Session. Okta will drop a session cookie called _sid_ in the domain of your Okta org:
__Postman Request: _sessionCookieRedirect___
```bash
{{url}}/login/sessionCookieRedirect?token={{sessionToken}}&redirectUrl={{redirectUrl}}
```
Where _sessionToken_ is the sessionToken value received in the response to _Primary Authentication_ /authn, and _redirectUrl_ is where Okta should redirect the user after establishing a session. Typically, this is a landing page or back to your custom sign-in page. Note the requirements listed here: https://developer.okta.com/use_cases/authentication/session_cookie#retrieving-a-session-cookie-by-visiting-a-session-redirect-link.

__Note:__ In order to check for an active session, you can call the /api/v1/sessions/me endpoint (see _Get Current Session_ below).

####status: MFA_REQUIRED

If a policy rule requires MFA, the response from _Primary Authentication_ will indicate status of __MFA_REQUIRED__, and will include a list of any MFA factors that the user has already enrolled.
```json
{
    "stateToken": "00zvsJ80BHTgWYo8iVtRdfEUNrqpZsSx4w-DLebNf3",
    "expiresAt": "2018-10-03T20:30:06.000Z",
    "status": "MFA_REQUIRED",
    "_embedded": {
        "user": {
            "id": "00u2e9pzr0aCdtXuI1t3",
            "passwordChanged": "2017-05-24T19:52:29.000Z",
            "profile": {
                "login": "jeff.beck@acme.com",
                "firstName": "Jeff",
                "lastName": "Beck",
                "locale": "en",
                "timeZone": "America/Los_Angeles"
            }
        },
        "factors": [
            {
                "id": "sms2s9jimvGguB7zY1t7",
                "factorType": "sms",
                "provider": "OKTA",
                "vendorName": "OKTA",
                "profile": {
                    "phoneNumber": "+1 XXX-XXX-4259"
                },
                "_links": {
                    "verify": {
                        "href": "https://oktalane.okta.com/api/v1/authn/factors/sms2e9jimvGguxaxat7/verify",
                        "hints": {
                            "allow": [
                                "POST"
                            ]
                        }
                    }
                }
            },
            {
                "id": "ufs2e9klmhQWN7jX51t7",
                "factorType": "question",
                "provider": "OKTA",
                "vendorName": "OKTA",
                "profile": {
                    "question": "name_of_first_plush_toy",
                    "questionText": "What is the name of your first stuffed animal?"
                },
                "_links": {
                    "verify": {
                        "href": "https://oktalane.okta.com/api/v1/authn/factors/ufs2e9klmhQWN7xaxaxa7/verify",
                        "hints": {
                            "allow": [
                                "POST"
                            ]
                        }
                    }
                }
            },
            {
                "id": "uft4ox35mszVJHRg21t7",
                "factorType": "token:software:totp",
                "provider": "GOOGLE",
                "vendorName": "GOOGLE",
                "profile": {
                    "credentialId": "jeff.beck@acme.com"
                },
                "_links": {
                    "verify": {
                        "href": "https://oktalane.okta.com/api/v1/authn/factors/uft4ox35msxaxaxat7/verify",
                        "hints": {
                            "allow": [
                                "POST"
                            ]
                        }
                    }
                }
            }
        ],
        "policy": {
            "allowRememberDevice": false,
            "rememberDeviceLifetimeInMinutes": 0,
            "rememberDeviceByDefault": false,
            "factorsPolicyInfo": {}
        }
    },
    "_links": {
        "cancel": {
            "href": "https://oktalane.okta.com/api/v1/authn/cancel",
            "hints": {
                "allow": [
                    "POST"
                ]
            }
        }
    }
}
```

###Challenge for MFA

For this example, we will challenge our user to verify an SMS challenge. 

####Send a new SMS challenge

__Postman Request: _Verify SMS Factor (new OTP challenge)___
```bash
curl -X POST \
  https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7/factors/sms2e9jimvGguxaxat7/verify \
  -H 'Accept: application/json' \
  -H 'Authorization: SSWS 009i9BR9qxaxaxaxaxaxasrHeTrHGef-l3UvGxaxarHGef' \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: 00168423-58b0-4e03-aeb2-826857303fab' \
  -d '{
}  '
```
Where _factorId_ is the ID of the SMS factor from the /authn response above.  We will receive a response similar to below, and the user will receive an SMS message with a verification code at the phone number they registered when enrolling the SMS factor.
```json
{
    "factorResult": "CHALLENGE",
    "profile": {
        "phoneNumber": "+15052694259"
    },
    "_links": {
        "verify": {
            "href": "https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7/factors/sms2e9jimvGguxaxat7/verify",
            "hints": {
                "allow": [
                    "POST"
                ]
            }
        },
        "factor": {
            "href": "https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7/factors/sms2e9jimvGguxaxat7",
            "hints": {
                "allow": [
                    "GET",
                    "DELETE"
                ]
            }
        }
    }
}
```

####Verify the SMS

To verify the SMS challenge, call the same API but include the verification code provided by the user:
__Postman Request: _Verify SMS Factor (verify token)___

```bash
curl -X POST \
  https://oktalane.okta.com/api/v1/authn/factors/sms2e9jimvGguxaxat7/verify \
  -H 'Accept: application/json' \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: 194e7126-fe0b-4c36-a114-e2129c538f70' \
  -d '{
	"stateToken": "00zvsJ80BHTgWYo8iVtRdfEUNrqpZsSx4w-DLebNf3",
	"passCode": "431261"
}'
```
If the verification succeeds, our response will include a sessionToken that can be exchanged for an Okta session as described above in the _Primary Authentication_ section.
```bash
{
    "expiresAt": "2018-10-03T20:52:59.000Z",
    "status": "SUCCESS",
    "sessionToken": "20111cnsvjlsAvS1drP_rV3pHFkVD2AfEupFMVvMf62B2wPQdvRo5Fx",
    "_embedded": {
        "user": {
            "id": "00u2e9hju0dCitSuI1t7",
            "passwordChanged": "2017-05-24T19:52:29.000Z",
            "profile": {
                "login": "jeff.beck@acme.com",
                "firstName": "Jeff",
                "lastName": "Beck",
                "locale": "en",
                "timeZone": "America/Los_Angeles"
            }
        }
    }
}
```

###Enroll an MFA factor

This example shows how to enroll Google Authenticator as an MFA factor.
__Postman Request: _Enroll Google Authenticator Factor___
```bash
curl -X POST \
  https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7/factors \
  -H 'Accept: application/json' \
  -H 'Authorization: SSWS 009i9BR9qxaxaxaxaxaxasrHeTrHGef-l3UvGxaxarHGef' \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: fd9598e7-6022-4b23-a923-efcb694b836e' \
  -d '{
  "factorType": "token:software:totp",
  "provider": "GOOGLE"
}'
```
The response will include a link to the QR code to display to the user:
```bash
{
    "id": "uft8gh35mszUCXKg21t7",
    "factorType": "token:software:totp",
    "provider": "GOOGLE",
    "vendorName": "GOOGLE",
    "status": "PENDING_ACTIVATION",
    "created": "2018-10-02T21:28:08.000Z",
    "lastUpdated": "2018-10-02T21:28:08.000Z",
    "profile": {
        "credentialId": "jeff.beck@acme.com"
    },
    "_links": {
        "activate": {
            "href": "https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7/factors/uft8gh35msxaxaxa21t7/lifecycle/activate",
            "hints": {
                "allow": [
                    "POST"
                ]
            }
        },
        "self": {
            "href": "https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7/factors/uft8gh35msxaxaxa21t7",
            "hints": {
                "allow": [
                    "GET"
                ]
            }
        },
        "user": {
            "href": "https://oktalane.okta.com/api/v1/users/uft8gh35msxaxaxa21t7",
            "hints": {
                "allow": [
                    "GET"
                ]
            }
        }
    },
    "_embedded": {
        "activation": {
            "timeStep": 30,
            "sharedSecret": "UNEAPF65QJPOTJ2P",
            "encoding": "base32",
            "keyLength": 6,
            "factorResult": "WAITING",
            "_links": {
                "qrcode": {
                    "href": "https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7/factors/uft8gh35msxaxaxa21t7/qr/201114UIGNtbL-hUn4wVUJsESe1QBrLgKC7QlRVv_axN-xaxaxaxaxaxa",
                    "type": "image/png"
                }
            }
        }
    }
}
```
The user will scan the QR code with the Google Authenticator app, after which they can provide the OTP code to Okta to activate the factor.
__Postman Request: _Activate TOTP Factor___
```bash
curl -X POST \
  https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7/factors/sms2e9jimvGxaxaY1t7/lifecycle/activate \
  -H 'Accept: application/json' \
  -H 'Authorization: SSWS 009i9BR9qxaxaxaxaxaxasrHeTrHGef-l3UvGxaxarHGef' \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: e2cf3e17-efa7-46cb-8295-0735ac0f9018' \
  -d '{
  "passCode": "739841"
}  '
```
Response:
```bash
{
    "id": "uft8he99pkllUCXKg21t7",
    "factorType": "token:software:totp",
    "provider": "GOOGLE",
    "vendorName": "GOOGLE",
    "status": "ACTIVE",
    "created": "2018-10-02T21:28:08.000Z",
    "lastUpdated": "2018-10-02T21:29:58.000Z",
    "profile": {
        "credentialId": "jeff.beck@acme.com"
    },
    "_links": {
        "self": {
            "href": "https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7/factors/uft8he99pkllUCXKg21t7",
            "hints": {
                "allow": [
                    "GET",
                    "DELETE"
                ]
            }
        },
        "verify": {
            "href": "https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7/factors/uft8he99pkllUCXKg21t7/verify",
            "hints": {
                "allow": [
                    "POST"
                ]
            }
        },
        "user": {
            "href": "https://oktalane.okta.com/api/v1/users/uft8he99pkllUCXKg21t7",
            "hints": {
                "allow": [
                    "GET"
                ]
            }
        }
    }
}
```

### Get the current session

__Postman Request: _Get Current Session___
```bash
curl -X GET \
  https://oktalane.okta.com/api/v1/sessions/me \
  -H 'Cache-Control: no-cache' \
  -H 'Postman-Token: cefc7d11-8033-4f43-b4f7-25ddba8a3373'
```
If there is a current session,  you will receive a response like:
```json
{
    "id": "102zYtGFjiaSHCussAtVkSkeg",
    "userId": "00u2e9hju0dCitSuI1t7",
    "login": "jeff.beck@acme.com",
    "createdAt": "2018-10-03T20:54:16.000Z",
    "expiresAt": "2018-10-03T22:59:14.000Z",
    "status": "ACTIVE",
    "lastPasswordVerification": "2018-10-03T20:54:16.000Z",
    "lastFactorVerification": "2018-10-03T20:53:03.000Z",
    "amr": [
        "pwd"
    ],
    "idp": {
        "id": "00o2bioj34AlsZYJf1t7",
        "type": "OKTA"
    },
    "mfaActive": true,
    "_links": {
        "self": {
            "href": "https://oktalane.okta.com/api/v1/sessions/me",
            "hints": {
                "allow": [
                    "GET",
                    "DELETE"
                ]
            }
        },
        "refresh": {
            "href": "https://oktalane.okta.com/api/v1/sessions/me/lifecycle/refresh",
            "hints": {
                "allow": [
                    "POST"
                ]
            }
        },
        "user": {
            "name": "Jeff Beck",
            "href": "https://oktalane.okta.com/api/v1/users/me",
            "hints": {
                "allow": [
                    "GET"
                ]
            }
        }
    }
}
```
You should now see GOOGLE listed as an enrolled factor if you call the /factors endpoint:
```bash
curl -X GET \
  https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7/factors \
  -H 'Accept: application/json' \
  -H 'Authorization: SSWS 009i9BR9qxaxaxaxaxaxasrHeTrHGef-l3UvGxaxarHGef' \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: 570602cf-9167-4b79-9e93-cbc0d8d03345'
```

```json
[
    {
        "id": "ufs2e9klmhQWN7jX51t7",
        "factorType": "question",
        "provider": "OKTA",
        "vendorName": "OKTA",
        "status": "ACTIVE",
        "created": "2017-05-24T19:52:42.000Z",
        "lastUpdated": "2017-05-24T19:52:42.000Z",
        "profile": {
            "question": "name_of_first_plush_toy",
            "questionText": "What is the name of your first stuffed animal?"
        },
        "_links": {
            "questions": {
                "href": "https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7/factors/questions",
                "hints": {
                    "allow": [
                        "GET"
                    ]
                }
            },
            "self": {
                "href": "https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7/factors/ufs2e9klmhQWN7jX51t7",
                "hints": {
                    "allow": [
                        "GET",
                        "DELETE"
                    ]
                }
            },
            "user": {
                "href": "https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7",
                "hints": {
                    "allow": [
                        "GET"
                    ]
                }
            }
        }
    },
    {
        "id": "sms2e9jimvGguM6zY1t7",
        "factorType": "sms",
        "provider": "OKTA",
        "vendorName": "OKTA",
        "status": "ACTIVE",
        "created": "2017-05-24T19:53:13.000Z",
        "lastUpdated": "2017-05-24T19:53:13.000Z",
        "profile": {
            "phoneNumber": "+15052694259"
        },
        "_links": {
            "self": {
                "href": "https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7/factors/sms2e9jimvGguM6zY1t7",
                "hints": {
                    "allow": [
                        "GET",
                        "DELETE"
                    ]
                }
            },
            "verify": {
                "href": "https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7/factors/sms2e9jimvGguM6zY1t7/verify",
                "hints": {
                    "allow": [
                        "POST"
                    ]
                }
            },
            "user": {
                "href": "https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7",
                "hints": {
                    "allow": [
                        "GET"
                    ]
                }
            }
        }
    },
    {
        "id": "uft8gh35mszUCXKg21t7",
        "factorType": "token:software:totp",
        "provider": "GOOGLE",
        "vendorName": "GOOGLE",
        "status": "ACTIVE",
        "created": "2018-10-02T21:28:08.000Z",
        "lastUpdated": "2018-10-02T21:29:58.000Z",
        "profile": {
            "credentialId": "jeff.beck@acme.com"
        },
        "_links": {
            "self": {
                "href": "https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7/factors/uft8gh35mszUCXKg21t7",
                "hints": {
                    "allow": [
                        "GET",
                        "DELETE"
                    ]
                }
            },
            "verify": {
                "href": "https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7/factors/uft8gh35mszUCXKg21t7/verify",
                "hints": {
                    "allow": [
                        "POST"
                    ]
                }
            },
            "user": {
                "href": "https://oktalane.okta.com/api/v1/users/00u2e9hju0dCitSuI1t7",
                "hints": {
                    "allow": [
                        "GET"
                    ]
                }
            }
        }
    }
]
```

##User Registration

###Create a user with a password

This request is typical for registering new users into a custom application. The registration form should collect the user's first and last name, primary email address, and preferred username (email format is not required), as well as the user's password. The _activate=true_ query parameter will cause this user to be created in Okta as in an active status.

```bash
curl -X POST \
  'https://btgapi.okta.com/api/v1/users?activate=true' \
  -H 'Accept: application/json' \
  -H 'Authorization: SSWS 0084-m5qg88tdJln8L4b6L94pXmLQmwFSAzWWTg_fu' \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: 23972c92-8ea6-480f-a343-a637be6c03d9' \
  -d '{
  "profile": {
    "firstName": "Bruce",
    "lastName": "Banner",
    "email": "bruce@amce.com",
    "login": "bruce@amce.com",
    "mobilePhone": "5052694259"
  },
  "credentials": {
    "password" : { "value": "D3m0demo1" }
  },
  "groupIds": [
  	"00g11eyizklnBxpN42p7",
  	"00g11dsz2ebabDtTI2p7"
  	]
}'
```
__Note:__The request above includes some attributes, such as mobilePhone and the groupIds array, which represents the Okta internal IDs of groups to which this user will automatically be added. Group membership can, in turn, be used by Okta policies (such as determining which users will be prompted for MFA and when), application assignment, etc.

A successful response looks like this:

```json
{
    "id": "00u28o8b4vIBpE4Cd2p7",
    "status": "ACTIVE",
    "created": "2018-10-04T20:58:17.000Z",
    "activated": "2018-10-04T20:58:18.000Z",
    "statusChanged": "2018-10-04T20:58:18.000Z",
    "lastLogin": null,
    "lastUpdated": "2018-10-04T20:58:18.000Z",
    "passwordChanged": "2018-10-04T20:58:18.000Z",
    "profile": {
        "firstName": "Bruce",
        "lastName": "Banner",
        "mobilePhone": "5052694259",
        "secondEmail": null,
        "login": "bruce@amce.com",
        "email": "bruce@amce.com"
    },
    "credentials": {
        "password": {},
        "emails": [
            {
                "value": "bruce@amce.com",
                "status": "VERIFIED",
                "type": "PRIMARY"
            }
        ],
        "provider": {
            "type": "OKTA",
            "name": "OKTA"
        }
    },
    "_links": {
        "suspend": {
            "href": "https://btgapi.okta.com/api/v1/users/00u28o8b4vIBpE4Cd2p7/lifecycle/suspend",
            "method": "POST"
        },
        "resetPassword": {
            "href": "https://btgapi.okta.com/api/v1/users/00u28o8b4vIBpE4Cd2p7/lifecycle/reset_password",
            "method": "POST"
        },
        "forgotPassword": {
            "href": "https://btgapi.okta.com/api/v1/users/00u28o8b4vIBpE4Cd2p7/credentials/forgot_password",
            "method": "POST"
        },
        "expirePassword": {
            "href": "https://btgapi.okta.com/api/v1/users/00u28o8b4vIBpE4Cd2p7/lifecycle/expire_password",
            "method": "POST"
        },
        "changeRecoveryQuestion": {
            "href": "https://btgapi.okta.com/api/v1/users/00u28o8b4vIBpE4Cd2p7/credentials/change_recovery_question",
            "method": "POST"
        },
        "self": {
            "href": "https://btgapi.okta.com/api/v1/users/00u28o8b4vIBpE4Cd2p7"
        },
        "changePassword": {
            "href": "https://btgapi.okta.com/api/v1/users/00u28o8b4vIBpE4Cd2p7/credentials/change_password",
            "method": "POST"
        },
        "deactivate": {
            "href": "https://btgapi.okta.com/api/v1/users/00u28o8b4vIBpE4Cd2p7/lifecycle/deactivate",
            "method": "POST"
        }
    }
}
```

###Auto-enrolling SMS factor

If your registration form collects the users mobile number, you can auto-enroll that number for SMS Multi-factor Authentication: https://developer.okta.com/docs/api/resources/factors#enroll-and-auto-activate-okta-sms-factor

### User Credential Operations

Documentation for the APIs needed to handle forgotten password, changing password, etc. can be found here: https://developer.okta.com/docs/api/resources/users#credential-operations.

