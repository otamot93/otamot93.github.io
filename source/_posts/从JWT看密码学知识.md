---
title: 从JWT看密码学知识
date: 2025-12-22 22:09:07
tags:
    - cryptography
---


JWT（JSON Web Token）常被当作“登录态”的替代品：服务端签发一个紧凑字符串，客户端带着它到处跑，服务端不存 Session 也能完成鉴权，常用语。但如果把 JWT 拆开看，会发现它包含了很多密码学知识：**哈希、消息认证码、数字签名、证书与格式**……这篇文章就以 JWT 为主线，介绍下相关的密码学知识。

---

<!--more-->

## 1. 为什么需要 JWT

传统 Session 方案的核心问题，在分布式时代会被放大：

- **扩展性问题**：Session 需要服务器端存储；多实例部署要共享或粘滞会话，横向扩展困难。
- **跨端/跨域限制**：Cookie 受同源策略等限制；移动端、小程序、SPA 往往更希望走 Header。
- **性能瓶颈**：每次请求都查 Session 存储（缓存/数据库），网络与 IO 开销不可忽视，特别是高并发的场景下，JWT在gateway上被验证了，也不需要依赖redis等存储。

JWT 选择了一条很“工程化”的路：把必要的上下文放进令牌里，让服务端用密码学手段验证它是否可信，从而实现**无状态扩展**。典型场景包括微服务、API 网关、移动端/小程序与 SPA。

---

## 2. JWT 的本质与三段式结构

JWT 最常见的是 **JWS（签名型 JWT）**，形态如下：

```
base64url(header) . base64url(payload) . base64url(signature)
```

- **Header（元信息）**：声明签名算法 `alg`、类型 `typ`，可选 `kid`（Key ID）用于多密钥环境快速定位公钥。
- **Payload（载荷/声明 Claims）**：包含标准声明与业务数据，例如：
  - 标准：`iss`（签发者）、`aud`（受众）、`sub`（主体）、`exp`（过期时间）、`iat`（签发时间）、`nbf`（生效时间）、`jti`（唯一 ID）。
  - 业务自定义：`roles`、`tenant_id` 等。
- **Signature（签名）**：对 `base64url(header) + "." + base64url(payload)` 这个串做“可验证的认证”，从而提供**完整性与来源可信**。

一个关键点：**Base64url 是编码，不是加密**。Header 与 Payload 默认都是可读的。

**代码示例：手动解码 JWT 结构**

```python
import base64
import json

token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IuW8oOS4iSIsImlhdCI6MTUxNjIzOTAyMn0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c"

# 分割三段
header_b64, payload_b64, signature_b64 = token.split(".")

# Base64url 解码（需要补齐 padding）
def b64url_decode(data):
    padding = 4 - len(data) % 4
    if padding != 4:
        data += "=" * padding
    return base64.urlsafe_b64decode(data)

# 解码 Header 和 Payload
header = json.loads(b64url_decode(header_b64))
payload = json.loads(b64url_decode(payload_b64))

print(f"Header:  {json.dumps(header, indent=2)}")
print(f"Payload: {json.dumps(payload, indent=2, ensure_ascii=False)}")
print(f"Signature (raw bytes): {b64url_decode(signature_b64).hex()}")
```

输出：
```
Header:  {
  "alg": "HS256",
  "typ": "JWT"
}
Payload: {
  "sub": "1234567890",
  "name": "张三",
  "iat": 1516239022
}
Signature (raw bytes): 49f94ac7044948c78a285d904f87f0a4c7897f7e8f3a4eb2255fda750b2cc397
```

---

## 3. JWT 的安全边界

围绕 JWT 最容易踩的坑，恰好能帮助我们划清密码学边界：

1. **JWT 默认不加密**：Payload 只是 Base64url 编码，任何拿到 token 的人都能解码看到内容；不要把敏感信息（身份证号、银行卡、明文权限策略等）放进去。
2. **Base64 不是安全手段**：它不提供保密性，必须依赖 **HTTPS/TLS** 保护传输链路。
3. **签名防篡改，但不防泄露**：签名保证“没被改、来自持钥者”，不保证“别人看不到”。
4. **密钥管理决定上限**：对称密钥泄露（如 HS256 secret）会导致全局信任崩塌；非对称能更好地做“签发/验证分离”。
5. **必须禁用 `none`**：历史上出现过“`alg=none`”绕过验签的漏洞；服务端必须强制校验并限制允许的算法集合。
6. **严格校验头部字段**：不要“信 token 里说自己用什么算法”；应由服务端配置决定允许的 `alg`，并配合 `kid` 做安全的密钥选择。

---

## 4. 从 JWT 看密码学原语

JWT 签名保证了令牌的完整性和确认令牌确实由可信方签发，签名不保障保密性，payload可以被读取，不要把机密的内容放到payload里，可以把敏感内容放到服务端，用token去获取。JWT的密码学原语主要包含:哈希、hmac、数字签名

### 4.1 摘要（哈希）

哈希（摘要、数字指纹）把任意长度输入映射为固定长度输出（如 SHA-256 输出 256-bit），典型性质包括：

- **固定长度输出**：便于进入签名/认证流程。但也不是绝对的，可扩展输出函数（XOF），输出长度是可以按需指定的，比如SHAKE128/256，BLAKE3等
- **雪崩效应**：输入微小变化会导致输出巨大变化，便于检测篡改。
- **不可逆性**：无法从摘要反推出原文。
- **抗碰撞性**：难以找到不同输入产生相同摘要（攻击者难以"伪造同摘要内容"）。

哈希不仅用于 JWT，也用于密码存储（更常见的是 bcrypt/Argon2 这类"密码派生函数"）、文件校验、区块链、匿名化等。


**代码示例：SHA-256 哈希计算**

```python
import hashlib

# 计算字符串的 SHA-256 哈希
message = "Hello, JWT!"
hash_digest = hashlib.sha256(message.encode()).hexdigest()
print(f"原文: {message}")
print(f"SHA-256: {hash_digest}")
# 输出: SHA-256: 7f83b1657ff1fc53b92dc18148a1d65dfc2d4b1fa3d677284addd200126d9069

# 演示雪崩效应：改一个字符，哈希完全不同
message2 = "Hello, JWT?"
hash_digest2 = hashlib.sha256(message2.encode()).hexdigest()
print(f"\n原文: {message2}")
print(f"SHA-256: {hash_digest2}")
# 输出: SHA-256: bba636fe98001c0054313ecf35c993e67cde5f835e30a2602bdf669e168024dc
```

### 4.2 对称的可认证验证-HMAC

HMAC（Hash-based Message Authentication Code）用**同一把共享密钥**，在消息上计算认证码（MAC）：

```
mac = HMAC_SHA256(secret, message)
```

验证方用同一密钥复算并比对，一致则说明：

- 消息**未被篡改**（完整性）
- 消息来自**持有密钥的一方**（来源认证）

**代码示例：Python 实现 HMAC-SHA256**

```python
import hmac
import hashlib

secret = b"my-secret-key"
message = b"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0"

# 计算 HMAC-SHA256
mac = hmac.new(secret, message, hashlib.sha256).digest()

# 验证：重新计算并比对（使用 hmac.compare_digest 防止时序攻击）
received_mac = mac  # 假设这是收到的 MAC
is_valid = hmac.compare_digest(
    hmac.new(secret, message, hashlib.sha256).digest(),
    received_mac
)
print(f"验证结果: {is_valid}")  # True
```

#### 4.2.1 HMAC和salt+SHA-256差异
两者很像，都是利用了sha256函数，同样存在要保护内容（secret和salt）。核心区别是HMAC是专门设计的“带密钥的消息认证码”，而salt+SHA-256常用于密码存储的去重和防彩虹表（推荐也用Argon2/bcrypt）。

### 4.3 数字签名

 数字签名是一种用非对称密码实现“身份认证 + 完整性校验 + 不可否认性”的机制。

- **私钥签名**：只有签发方持有，必须严密保护
- **公钥验签**：可以广泛分发，任何验证方都能用它验证签名

这非常适合"一个身份中心签发 token、多服务验签"的架构。
常见算法：RSA‑PSS、ECDSA、Ed25519。

**代码示例：Python 实现 RSA 数字签名

```python
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import rsa, padding

# 生成 RSA 密钥对
private_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
public_key = private_key.public_key()

message = b"eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0"

# 私钥签名
signature = private_key.sign(
    message,
    padding.PKCS1v15(),  # RS256 使用 PKCS#1 v1.5
    hashes.SHA256()
)

# 公钥验签
try:
    public_key.verify(signature, message, padding.PKCS1v15(), hashes.SHA256())
    print("签名验证成功 ✓")
except Exception as e:
    print(f"签名验证失败: {e}")
```

---

## 5. JWA 算法矩阵

JWT 相关算法在 JWA（JSON Web Algorithms）里按密钥形态与数学基础分族：

### 5.1 HS256/384/512：对称 HMAC

- **优点**：部署简单、吞吐高；适合单体应用或可信内网。
- **风险**：密钥一旦泄露，攻击者可伪造任意 token；密钥轮换与最短过期时间非常关键。

HS256 的核心就是对 `header.payload` 做 `HMAC-SHA256`。

**代码示例：HS256 完整 JWT 签发与验证**

```python
import jwt
import datetime

secret = "your-256-bit-secret"  # 生产环境应使用足够长的随机密钥

# 签发 JWT
payload = {
    "sub": "1234567890",
    "name": "张三",
    "iat": datetime.datetime.now(datetime.timezone.utc),
    "exp": datetime.datetime.now(datetime.timezone.utc) + datetime.timedelta(hours=1)
}
token = jwt.encode(payload, secret, algorithm="HS256")
print(f"Token: {token}")

# 验证 JWT（指定允许的算法，防止算法混淆攻击）
try:
    decoded = jwt.decode(token, secret, algorithms=["HS256"])
    print(f"验证成功: {decoded}")
except jwt.ExpiredSignatureError:
    print("Token 已过期")
except jwt.InvalidTokenError as e:
    print(f"无效 Token: {e}")

# 输出
# Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6Ilx1NWYyMFx1NGUwOSIsImlhdCI6MTc2NjQxNzA2MiwiZXhwIjoxNzY2NDIwNjYyfQ.khids45iOQ0TGgTPP8mtxclbb93FVMskPQRlfJMxtjk
# 验证成功: {'sub': '1234567890', 'name': '张三', 'iat': 1766417062, 'exp': 1766420662}
```

### 5.2 RS256/PS256

RS256 和 PS256（**推荐**） 都是基于 RSA 的签名算法，区别在填充方式与安全性：

  - RS256 = RSA‑PKCS#1 v1.5 + SHA‑256，老算法、兼容性好，但填充方式较旧。
  - PS256 = RSA‑PSS + SHA‑256，现代算法，带随机盐，抗攻击性更强，推荐新系统使用。

RSA 的安全性建立在**大整数分解困难**上：已知大素数 `p,q` 计算 `n=p*q` 很容易，但给定足够大的 `n` 反推出 `p,q` 极难。

在 JWT 场景里的应用是“签名/验签”

- 先对 `header.payload` 做 SHA-256（摘要）
- 再按标准（如 PKCS#1 v1.5 或 PSS）构造待签名结构
- 私钥做模幂运算得到签名；公钥恢复并比对摘要，确认完整性与真实性

![](从JWT看密码学知识/signature.png)


**代码示例：RS256 JWT 签发与验证**

```python
import jwt
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives import serialization

# 生成 RSA 密钥对
private_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
public_key = private_key.public_key()

# 导出为 PEM 格式（实际项目中密钥应安全存储）
private_pem = private_key.private_bytes(
    encoding=serialization.Encoding.PEM,
    format=serialization.PrivateFormat.PKCS8,
    encryption_algorithm=serialization.NoEncryption()
)
public_pem = public_key.public_bytes(
    encoding=serialization.Encoding.PEM,
    format=serialization.PublicFormat.SubjectPublicKeyInfo
)

# 签发 JWT（使用私钥）
payload = {"sub": "1234567890", "role": "admin"}
token = jwt.encode(payload, private_pem, algorithm="RS256")
print(f"RS256 Token: {token[:50]}...")

# 验证 JWT（使用公钥 - 可分发给任意验证服务）
decoded = jwt.decode(token, public_pem, algorithms=["RS256"])
print(f"验证成功: {decoded}")
```


### 5.3 ES256/384/512：ECDSA（椭圆曲线签名）

ECDSA 的优势在于：在相同安全级别下，**密钥更短、签名更小、性能更好**。例如常见对比：

- 约 **128-bit 安全强度**：RSA 需要约 **3072-bit**，而 ECDSA（P-256）只要 **256-bit**。

因此 ES256 很适合移动端、带宽敏感、高并发的分布式系统。

**代码示例：ES256 JWT 签发与验证**

```python
import jwt
from cryptography.hazmat.primitives.asymmetric import ec
from cryptography.hazmat.primitives import serialization

# 生成 ECDSA 密钥对（P-256 曲线，对应 ES256）
private_key = ec.generate_private_key(ec.SECP256R1())
public_key = private_key.public_key()

# 导出为 PEM 格式
private_pem = private_key.private_bytes(
    encoding=serialization.Encoding.PEM,
    format=serialization.PrivateFormat.PKCS8,
    encryption_algorithm=serialization.NoEncryption()
)
public_pem = public_key.public_bytes(
    encoding=serialization.Encoding.PEM,
    format=serialization.PublicFormat.SubjectPublicKeyInfo
)

# 签发 JWT
payload = {"sub": "user123", "permissions": ["read", "write"]}
token = jwt.encode(payload, private_pem, algorithm="ES256")
print(f"ES256 Token: {token[:50]}...")

# 验证 JWT
decoded = jwt.decode(token, public_pem, algorithms=["ES256"])
print(f"验证成功: {decoded}")
```


---

## 6. JWT密钥管理与工程最佳实践

JWT 的密码学原语再正确，也架不住“密钥与工程细节”出问题。实践建议可以按三个层次落地：

### 6.1 服务端校验策略（防绕过）

- **白名单算法**：服务端配置允许的 `alg`，拒绝一切未知/不期望算法；明确禁用 `none`。
- **固定算法与密钥类型绑定**：例如只允许 `RS256/PS256` 且只接受 RSA 公钥；避免“算法混淆”。
- **严格校验 Claims**：至少校验 `exp`；按需要校验 `iss/aud/nbf`；必要时使用 `jti` 做幂等与黑名单。

### 6.2 过期与撤销（无状态的代价）

JWT 无状态带来一个天然难题：**签发后很难立即撤销**。常见解法：

- **短过期时间 + 刷新机制**（Refresh Token/会话续期）
- 对关键场景引入**黑名单/版本号**（需要一定“有状态”存储）

### 6.3 密钥轮换与分发：`kid`、JWK 与 JWKS

多密钥环境通常用 `kid` 做定位，配合 JWKS 暴露公钥集合：

- **JWK（JSON Web Key）**：用 JSON 表示密钥，Web 原生友好。
- **JWKS（JWK Set）**：密钥集合，常通过 `/.well-known/jwks.json` 提供给验证方动态拉取。

这样可以做到：

- 平滑轮换：发布新公钥（新 `kid`）→ 签发新 token → 旧公钥保留一段时间用于验旧 token → 下线旧公钥。

**代码示例：生成 JWK 与 JWKS**

```python
import json
from cryptography.hazmat.primitives.asymmetric import rsa, ec
from cryptography.hazmat.primitives import serialization
import base64

def int_to_base64url(n, length=None):
    """将整数转换为 Base64url 编码"""
    if length is None:
        length = (n.bit_length() + 7) // 8
    data = n.to_bytes(length, byteorder='big')
    return base64.urlsafe_b64encode(data).rstrip(b'=').decode('ascii')

# 生成 RSA 密钥并转换为 JWK
private_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
public_key = private_key.public_key()
public_numbers = public_key.public_numbers()

jwk = {
    "kty": "RSA",
    "kid": "key-2024-01",  # 密钥 ID，用于轮换时标识
    "use": "sig",          # 用途：签名
    "alg": "RS256",
    "n": int_to_base64url(public_numbers.n),
    "e": int_to_base64url(public_numbers.e)
}

# JWKS 是 JWK 的集合
jwks = {"keys": [jwk]}

print("JWK (单个公钥):")
print(json.dumps(jwk, indent=2))
print("\nJWKS (公钥集合，可通过 /.well-known/jwks.json 暴露):")
print(json.dumps(jwks, indent=2))
```

**代码示例：从 JWKS URL 获取公钥验证 JWT**

```python
import jwt
import requests
from jwt import PyJWKClient

# 从 JWKS 端点获取公钥（以 Auth0 为例）
jwks_url = "https://your-tenant.auth0.com/.well-known/jwks.json"
jwks_client = PyJWKClient(jwks_url)

# 验证 JWT 时自动根据 kid 获取对应公钥
token = "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImtleS0yMDI0LTAxIn0..."
try:
    signing_key = jwks_client.get_signing_key_from_jwt(token)
    decoded = jwt.decode(
        token,
        signing_key.key,
        algorithms=["RS256"],
        audience="your-api-audience"
    )
    print(f"验证成功: {decoded}")
except jwt.exceptions.InvalidTokenError as e:
    print(f"验证失败: {e}")
```

---

## 7. 密钥格式与证书

做过一次“把公钥喂给网关/SDK”的人，往往会发现困难不在数学，而在格式与封装。

### 7.1 DER 与 PEM：编码与承载

- **DER**：ASN.1 的二进制编码，适合存储。
- **PEM**：把 DER 做 Base64 编码，并加上 `BEGIN ... END ...` 头尾，适合文本传输与配置。

### 7.2 PKCS#1 vs PKCS#8：内部结构差异

- **PKCS#1**：只为 RSA 定义的公钥/私钥结构；常见 PEM 头尾 `BEGIN RSA PRIVATE KEY`。
- **PKCS#8**：通用私钥封装（可容纳 RSA/EC 等），含算法 OID；常见 PEM 头尾 `BEGIN PRIVATE KEY` / `BEGIN ENCRYPTED PRIVATE KEY`。

工程建议：**新系统优先 PKCS#8**

**代码示例：密钥格式转换**

```python
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import rsa

# 生成密钥
private_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)

# 导出为 PKCS#1 格式（RSA 专用，旧格式）
pkcs1_pem = private_key.private_bytes(
    encoding=serialization.Encoding.PEM,
    format=serialization.PrivateFormat.TraditionalOpenSSL,  # PKCS#1
    encryption_algorithm=serialization.NoEncryption()
)
print("PKCS#1 格式（BEGIN RSA PRIVATE KEY）:")
print(pkcs1_pem.decode()[:100] + "...")

# 导出为 PKCS#8 格式（通用，推荐）
pkcs8_pem = private_key.private_bytes(
    encoding=serialization.Encoding.PEM,
    format=serialization.PrivateFormat.PKCS8,
    encryption_algorithm=serialization.NoEncryption()
)
print("\nPKCS#8 格式（BEGIN PRIVATE KEY）:")
print(pkcs8_pem.decode()[:100] + "...")

# 导出为加密的 PKCS#8（生产环境推荐）
encrypted_pem = private_key.private_bytes(
    encoding=serialization.Encoding.PEM,
    format=serialization.PrivateFormat.PKCS8,
    encryption_algorithm=serialization.BestAvailableEncryption(b"my-password")
)
print("\n加密的 PKCS#8 格式（BEGIN ENCRYPTED PRIVATE KEY）:")
print(encrypted_pem.decode()[:100] + "...")
```

**OpenSSL 密钥格式转换命令**

```bash
# PKCS#1 转 PKCS#8
openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt \
    -in rsa_pkcs1.pem -out rsa_pkcs8.pem

# PKCS#8 转 PKCS#1
openssl rsa -in rsa_pkcs8.pem -out rsa_pkcs1.pem

# 查看 PEM 文件类型
head -1 private_key.pem
# -----BEGIN RSA PRIVATE KEY----- 表示 PKCS#1
# -----BEGIN PRIVATE KEY----- 表示 PKCS#8
# -----BEGIN ENCRYPTED PRIVATE KEY----- 表示加密的 PKCS#8
```

### 7.3 X.509 证书：带“信任链”的公钥分发

X.509 把公钥、主体信息、有效期等打包，并由 CA 签名，广泛用于 TLS。

---

## 8. 从 JWT 出发，看研发中的密码学“全景”

把 JWT 放回更大的工程图景里，它只是“用户认证”这一格，密码学还在其他场景内有重要作用。

- **用户认证**：密码哈希（bcrypt/Argon2）+ JWT（HS/RS/ES）+ OAuth2/OpenID Connect。
- **数据加密**：AES-GCM、字段级加密、信封加密（KMS 管 DEK）。
- **通信安全**：HTTPS/TLS 1.3，必要时 mTLS（双向认证）。
- **完整性校验**：HMAC-SHA256 做 API 签名、文件校验、防篡改与防重放（timestamp + nonce）。
- **身份证明**：X.509、代码签名（APK/npm/jar）、Git 签名（GPG/SSH）。
- **密钥管理**：KMS/HSM/Vault、权限隔离、审计、轮换与应急吊销。
- **隐私计算**: Oblivious Transfer、PSI、PIR、机密计算


---

# 参考
- [JWT 签名算法 HS256、RS256 及 ES256 及密钥生成](https://thiscute.world/posts/jwt-algorithm-key-generation/)

## 附录

### 常用工具

- **jwt.io**: 在线 JWT 解码与调试工具
- **OpenSSL**: 密钥生成与格式转换的瑞士军刀



