SAA-C03 sigue siendo el examen vigente. IAM te va a aparecer en las cuatro dominios, no solo en "Secure Architectures".

## Lo que tienes que dominar de IAM

**1. Primitivos**
Usuarios, grupos, roles y políticas. 

Cuándo NO usar usuarios IAM (respuesta casi siempre: usa roles o IAM Identity Center). 

Root account: qué solo puede hacer root (cerrar cuenta, cambiar plan de soporte, cambiar email/nombre, restaurar política de bucket S3 mal puesta), 

MFA obligatorio, cero access keys.

**2. Los seis tipos de política** — esto es lo que más reprueba gente
Identity-based, resource-based, permissions boundaries, SCPs (y RCPs de Organizations), session policies, ACLs. Para el examen: qué hace cada una y cuál se usa en el escenario que te describen.

**3. Lógica de evaluación de permisos**
El orden completo: Deny explícito → SCP → boundary → session policy → identity/resource policy → deny implícito. Casos clave: acceso cross-account requiere permiso en **ambos** lados (identity policy en la cuenta A + resource policy en la cuenta B), excepto con IAM roles donde basta la trust policy + permiso para llamar `sts:AssumeRole`. Un SCP no otorga nada, solo limita.

**4. Anatomía de una policy**
`Version`, `Sid`, `Effect`, `Principal`, `Action`/`NotAction`, `Resource`/`NotResource`, `Condition`. Wildcards y formato de ARN. `NotAction` con `Deny` (patrón de whitelist de regiones). Variables de política (`${aws:username}`, `${aws:PrincipalTag/dept}`).

**5. Condition keys que sí salen en el examen**
`aws:SourceIp`, `aws:VpcSourceIp`, `aws:SourceVpce`, `aws:SecureTransport`, `aws:MultiFactorAuthPresent`, `aws:PrincipalOrgID`, `aws:PrincipalArn`, `aws:SourceArn`/`aws:SourceAccount` (confused deputy), `aws:RequestedRegion`, `aws:PrincipalTag`/`aws:ResourceTag`/`ec2:ResourceTag`, `s3:prefix`, `s3:x-amz-server-side-encryption`, `kms:ViaService`. Y los operadores: `StringEquals` vs `StringLike` vs `ArnLike`, y el sufijo `IfExists` con `Null`.

**6. Roles y STS**
Trust policy vs permission policy. `sts:AssumeRole`, `AssumeRoleWithSAML`, `AssumeRoleWithWebIdentity`, `GetSessionToken`, `GetFederationToken`. External ID (el patrón del third-party/confused deputy — sí sale). Role chaining y su límite de 1 hora. Duración máxima de sesión. Credenciales temporales vs access keys: por qué las temporales siempre ganan en el examen.

**7. Roles de servicio**
Instance profile para EC2 (y que IMDSv2 es la respuesta correcta para SSRF). Execution role de Lambda. Task role vs task execution role en ECS (los confunden mucho: el execution role es para jalar la imagen de ECR y escribir logs; el task role es para lo que hace tu código). Service-linked roles y por qué no los editas. `iam:PassRole` — cuándo se necesita y por qué es peligroso.

**8. Federación e identidad**
IAM Identity Center (ex AWS SSO) como respuesta por defecto para acceso humano multi-cuenta: permission sets, integración con AD/Okta/Entra. SAML 2.0 federation. OIDC federation (GitHub Actions, EKS IRSA / Pod Identity). Cognito User Pools vs Identity Pools — cuál da credenciales AWS (identity pools). AD Connector vs AWS Managed Microsoft AD vs Simple AD. IAM Roles Anywhere para workloads on-prem.

**9. Multi-cuenta y Organizations**
OUs, SCPs, RCPs, Control Tower, delegated administrator. Estrategia de cuentas (prod/dev/security/log-archive). `aws:PrincipalOrgID` para restringir buckets a tu organización. Permissions boundary para delegar la creación de roles a devs sin escalación de privilegios.

**10. Resource-based policies por servicio**
S3 bucket policy (+ Block Public Access, + por qué ACLs están deprecadas, + presigned URLs y los permisos que heredan), KMS key policy (la key policy es la autoridad final — IAM solo no basta), SQS, SNS, Lambda resource policy (`AddPermission` para API Gateway/S3/EventBridge), Secrets Manager, ECR, EFS file system policy, API Gateway resource policy. Cross-account con KMS: necesitas key policy + grant/IAM en la otra cuenta.

**11. ABAC vs RBAC**
Tag-based access control, session tags, transitive tags. Es la respuesta a "cómo escalo permisos sin crear 400 políticas".

**12. Cifrado y secretos (pegado a IAM)**
KMS: CMK gestionada por cliente vs gestionada por AWS, key rotation, multi-region keys, envelope encryption, grants vs key policy. Secrets Manager (rotación automática, más caro) vs SSM Parameter Store (SecureString, gratis en standard). Nunca credenciales en código/variables de entorno.

**13. Gobierno y auditoría**
IAM Access Analyzer (external access, unused access, policy validation, policy generation desde CloudTrail), credential report, Last Accessed / Access Advisor, CloudTrail (qué registra y qué no — data events de S3/Lambda están apagados por default), AWS Config rules, Security Hub, GuardDuty. Rotación de llaves.

**14. Anti-patrones que el examen castiga**
Access keys de larga vida, `*:*`, usuarios IAM para apps, compartir credenciales entre cuentas, root para operación diaria, NAT/VPC "resueltos" con permisos IAM.

Lo que **no** necesitas para SAA: memorizar el JSON de cada política de servicio ni la sintaxis exacta de SCPs complejas. Sí necesitas leer una policy y decir si permite o niega una acción.

---

## 10 ejercicios de Terraform

Todos con `data "aws_iam_policy_document"` en vez de JSON inline, `terraform plan` limpio, y `aws sts assume-role` real para validar. Idealmente dos cuentas (usa una cuenta sandbox y tu cuenta personal, o AWS Organizations con cuentas nuevas).

**1. Cross-account assume role con External ID**
Cuenta A (confiada) tiene un rol `AuditorRole`; cuenta B asume con `sts:AssumeRole` + external ID obligatorio. Valida que sin el external ID falle. Agrega condición de MFA y `aws:PrincipalOrgID`. Módulo reutilizable con variables para la lista de cuentas confiadas.

**2. OIDC para GitHub Actions — cero access keys**
Crea el `aws_iam_openid_connect_provider` de GitHub, un rol con trust policy que restrinja por `repo:owner/repo:ref:refs/heads/main` (no con `StringLike` abierto — ese es el bug clásico), y un pipeline que haga `terraform apply` desde Actions usando ese rol.

**3. Permissions boundary con delegación a developers**
Una policy boundary que limita a ciertos servicios y regiones. Un rol `DeveloperRole` que puede crear roles e instance profiles **solo si** adjunta esa boundary (condición `iam:PermissionsBoundary`) y no puede borrarla ni modificarla. Prueba que un dev no puede escalar a admin.

**4. ABAC completo con session tags**
Una sola policy que otorga acceso a recursos donde `aws:PrincipalTag/project == aws:ResourceTag/project`. Roles con tags, EC2/S3 etiquetados por proyecto, y política que niega crear recursos sin tag de proyecto (`aws:RequestTag`). Demuestra que agregar un proyecto nuevo no requiere tocar ninguna policy.

**5. S3 endurecido por bucket policy**
Bucket con: deny si `aws:SecureTransport == false`, deny si el principal no está en tu org (`aws:PrincipalOrgID`), deny PutObject sin SSE-KMS con tu key específica, Block Public Access activado, y acceso solo vía VPC endpoint (`aws:SourceVpce`). Después escribe un test que intente cada violación y confirme el 403.

**6. KMS multi-cuenta con envelope encryption**
CMK en cuenta de seguridad, key policy que delega a IAM de la cuenta de workload, grants para un rol de Lambda, condición `kms:ViaService` para que la key solo se use a través de S3. Bonus: multi-region key con réplica y failover.

**7. Rol de EC2 mínimo + IMDSv2 forzado**
Instance profile que solo permite `s3:GetObject` sobre un prefijo específico y `ssm:GetParameter` sobre una ruta específica. Launch template con `http_tokens = "required"` y `http_put_response_hop_limit = 1`. Usa Session Manager en lugar de llaves SSH — sin puerto 22 abierto.

**8. Guardrails de Organizations con SCPs**
Terraform sobre `aws_organizations_*`: OUs (Sandbox, Prod, Security), SCP que restringe regiones vía `NotAction`/`aws:RequestedRegion`, SCP que niega deshabilitar CloudTrail/Config/GuardDuty, SCP que niega el uso del root user. Aplica a la OU, no a cuentas sueltas. Ojo: pruébalo en una org de juguete, un SCP mal hecho te tranca la cuenta.

**9. Pipeline de validación de políticas**
Integra `aws accessanalyzer validate-policy` + Checkov o tfsec en pre-commit y en CI. Crea deliberadamente tres políticas malas (wildcard admin, bucket público, `iam:PassRole` sin scope) y confirma que el pipeline falla. Agrega un analyzer de acceso externo que reporte findings.

**10. Break-glass + detección (el capstone)**
Rol de emergencia con permisos de admin que solo se puede asumir con MFA, sesión de máximo 1 hora, y trust policy limitada a dos principals nombrados. EventBridge rule sobre CloudTrail que dispare SNS cuando alguien lo asuma. Conéctalo con lo anterior: todo el setup (org + SCPs + boundaries + break-glass) en un solo repo con módulos y backend remoto en S3 con state lock.

Si quieres hacerlos en orden, el camino natural es 1 → 3 → 4 → 7 → 5 → 6 → 2 → 8 → 9 → 10.

Te lo puedo armar como documento vivo con checkboxes si prefieres irlo marcando.

Fuentes: [Guía de examen SAA-C03](https://docs.aws.amazon.com/aws-certification/latest/solutions-architect-associate-03/solutions-architect-associate-03.html), [Página de certificación AWS SAA](https://aws.amazon.com/certification/certified-solutions-architect-associate/)