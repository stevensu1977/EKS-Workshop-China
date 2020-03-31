# 步骤14-如何配置IngressController的认证
1. [nginx_ingress](https://kubernetes.github.io/ingress-nginx/examples/auth/basic/)
```yaml
kind: Ingress
metadata:
  name: nginx-ingress-with-auth
  annotations:
    # type of authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    # name of the secret that contains the user/password definitions
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    # message to display with an appropriate context why the authentication is required
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - foo'
```

2. [ALB_Ingress](https://github.com/kubernetes-sigs/aws-alb-ingress-controller/blob/master/docs/guide/ingress/annotation.md)
- auth-type: cognito
```yaml
kind: Ingress
metadata:
  labels:
    app.kubernetes.io/name: alb-ingress-controller
  name: alb-ingress-controller
  annotations:
    # Authentication type as cognito
    alb.ingress.kubernetes.io/auth-type: cognito
    # Specifies the set of user claims to be requested from the IDP
    alb.ingress.kubernetes.io/auth-scope: 'email openid'
    # Specifies the cognito idp configuration.
    alb.ingress.kubernetes.io/auth-idp-cognito: '{"UserPoolArn":"arn:aws:cognito-idp:us-west-2:xxx:userpool/xxx", "UserPoolClientId":"xxx", "UserPoolDomain":"xxx"}'
```

- auth-type: oidc
```yaml
kind: Ingress
metadata:
  labels:
    app.kubernetes.io/name: alb-ingress-controller
  name: alb-ingress-controller
  annotations:
    # Authentication type as oidc
    alb.ingress.kubernetes.io/auth-type: oidc
    # Specifies the set of user claims to be requested from the IDP
    alb.ingress.kubernetes.io/auth-scope: 'email openid'
    # Specifies the oidc idp configuration.
    alb.ingress.kubernetes.io/auth-idp-oidc: '{"Issuer":"xxx","AuthorizationEndpoint":"xxx","TokenEndpoint":"xxx","UserInfoEndpoint":"xxx","SecretName":"customizedSecretName"}'
```

# 参考资料

[Securing your k8s application using Ingress resource](https://medium.com/faun/securing-k8s-application-using-ingress-rule-nginx-ingress-controller-a819b0e11281)
