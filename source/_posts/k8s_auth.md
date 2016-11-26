---
title: kubernetes中authN/Z源码解析
tags:
	- kubernetes
	- go
---

* cmd/kube-apiserver/app/server.go
```
authorizer, err := apiserver.NewAuthorizerFromAuthorizationConfig(s.AuthorizationMode, s.AuthorizationPolicyFile)

authenticator, err := apiserver.NewAuthenticator(s.BasicAuthFile, s.ClientCAFile, s.TokenAuthFile, s.ServiceAccountKeyFile, s.ServiceAccountLookup, helper)

```
