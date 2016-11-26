---
title: k8s_auth
date: 2016-11-26 20:05:14
tags:
	- kubernetes
	- go
---

* cmd/kube-apiserver/app/server.go
```
authorizer, err := apiserver.NewAuthorizerFromAuthorizationConfig(s.AuthorizationMode, s.AuthorizationPolicyFile)

authenticator, err := apiserver.NewAuthenticator(s.BasicAuthFile, s.ClientCAFile, s.TokenAuthFile, s.ServiceAccountKeyFile, s.ServiceAccountLookup, helper)

```
