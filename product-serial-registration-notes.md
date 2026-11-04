# 시리얼 번호 보호 정리 (HMAC 지문 + AES-GCM 암호화)

> 개인 학습·발표용 메모입니다. `.gitignore`에 등록돼 있어 GitHub에 올라가지 않습니다.
> 코드를 읽어서 정리했고, 실행해 검증한 것은 아닙니다.

## 1. 한 줄 요약

시리얼 번호 원문은 DB에 저장하지 않습니다. 원문에서 **두 값을 따로** 만들어 저장합니다.

- **지문(HMAC)** → 중복 등록 방지 (되돌릴 수 없음)
- **암호문(AES)** → 원문 보호 + 필요할 때만 복원

```
        ┌─ HMAC ──→ 지문   (여기서 끝. 풀지 않음)
110112 ─┤
        └─ AES 암호화 ──→ 암호문 ──→ AES 복호화 ──→ 110112
```

두 작업은 서로 독립적이고, 둘 다 원문에서 출발합니다. HMAC으로 만든 값을 AES로 푸는 것이 아닙니다.

## 2. 용어

| 용어 | 뜻 |
| --- | --- |
| HMAC | 비밀 키 + 데이터로 고정 길이 값을 만드는 한 방향 계산. 같은 키·같은 입력이면 항상 같은 값, 거꾸로 되돌릴 수 없음 |
| 지문 | HMAC으로 만든 값 (이 문서 기준 이름) |
| AES-256-GCM | 키로 잠그고 같은 키로 푸는 대칭키 암호. GCM은 암호문 변조도 감지 |
| 암호화 / 복호화 | 원문을 잠그기 / 잠긴 값을 키로 풀어 원문으로 되돌리기 |
| IV | 암호화마다 새로 만드는 12바이트 난수. 같은 번호도 암호문이 매번 달라짐. 비밀이 아니라 암호문 앞에 붙여 저장 |

비유: 지문은 **믹서기로 간 주스**(재료로 되돌릴 수 없지만 색이 같은지는 비교 가능), 암호문은 **잠긴 상자**(열쇠가 있으면 열 수 있음).

## 3. `110112`가 저장되는 과정 (값은 설명용 예시)

| 컬럼 | 저장되는 값 |
| --- | --- |
| `serial_number_fingerprint` | `9f3a...c71e` (64자 hex) |
| `serial_number_ciphertext` | `k2Vx9...Qw==` (Base64) |
| 원문 | **없음** |

**같은 번호를 다시 등록하면**

- 지문: 똑같이 `9f3a...c71e` → DB에서 같은 값 발견 → 중복
- 암호문: IV가 달라 `p8Lm2...Zx==`처럼 전혀 다른 값 → 그래서 **비교용으로는 쓸 수 없고**, 비교 전용으로 지문을 따로 둠

| | 지문 (HMAC) | 암호문 (AES) |
| --- | --- | --- |
| 같은 번호를 넣으면 | 항상 같은 값 | 매번 다른 값 |
| 원문 복원 | 불가 | 키로 가능 |
| 역할 | 중복 확인 | 원문 보관·복원 |

원문을 저장해 두고 비교하는 것이 아니라, **새로 들어온 번호로 계산한 지문이 이미 저장된 지문과 같은지** 비교합니다.

## 4. 사용된 코드

### 4-1. 시리얼 번호 보호 클래스 — `product/service/SerialNumberProtector.java`

키 파생 (서버 시작 시). 마스터 시크릿에서 용도별 키 2개를 따로 만듭니다.

```java
public SerialNumberProtector(@Value("${app.product.serial-secret}") String secret) {
  if (secret == null || secret.length() < 32) {
    throw new IllegalStateException("app.product.serial-secret must be at least 32 characters");
  }
  byte[] master = secret.getBytes(StandardCharsets.UTF_8);
  this.fingerprintKey = new SecretKeySpec(derive(master, "serial-fingerprint"), "HmacSHA256");
  this.encryptionKey = new SecretKeySpec(derive(master, "serial-encryption"), "AES");
}
```

정규화. 공백·대소문자 차이로 중복 검사를 피하지 못하게 합니다.

```java
public static String normalize(String serialNumber) {
  return serialNumber.replaceAll("\\s+", "").toUpperCase(Locale.ROOT);
}
```

**HMAC — 지문 생성**

```java
public String fingerprint(String serialNumber) {
  return HexFormat.of().formatHex(hmac(fingerprintKey, normalize(serialNumber)));
}

private static byte[] hmac(SecretKeySpec key, String message) {
  Mac mac = Mac.getInstance("HmacSHA256");
  mac.init(key);
  return mac.doFinal(message.getBytes(StandardCharsets.UTF_8));
}
```

**AES — 암호화** (IV 12바이트를 앞에 붙여 Base64로 저장)

```java
public String encrypt(String serialNumber) {
  byte[] iv = new byte[IV_BYTES];
  secureRandom.nextBytes(iv);
  Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
  cipher.init(Cipher.ENCRYPT_MODE, encryptionKey, new GCMParameterSpec(TAG_BITS, iv));
  byte[] encrypted = cipher.doFinal(normalize(serialNumber).getBytes(StandardCharsets.UTF_8));
  byte[] combined = new byte[iv.length + encrypted.length];
  System.arraycopy(iv, 0, combined, 0, iv.length);
  System.arraycopy(encrypted, 0, combined, iv.length, encrypted.length);
  return Base64.getEncoder().encodeToString(combined);
}
```

**AES — 복호화**

```java
public String decrypt(String ciphertext) {
  byte[] combined = Base64.getDecoder().decode(ciphertext);
  Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
  cipher.init(Cipher.DECRYPT_MODE, encryptionKey, new GCMParameterSpec(TAG_BITS, combined, 0, IV_BYTES));
  return new String(cipher.doFinal(combined, IV_BYTES, combined.length - IV_BYTES), StandardCharsets.UTF_8);
}
```

(예외 처리는 생략하고 줄였습니다. 원본은 `GeneralSecurityException`을 `IllegalStateException`으로 감쌉니다.)

### 4-2. 호출하는 곳 — `product/service/ProductService.java`

**등록 `create`** — 지문으로 중복 확인, 암호문과 함께 저장

```java
@Transactional
public ProductResponse create(long ownerId, CreateProductRequest request) {
  ...
  String fingerprint = serialNumberProtector.fingerprint(request.serialNumber());   // HMAC
  if (productRepository.existsByManufacturerAndSerial(manufacturerNormalized, fingerprint)) {
    throw new CommonException(ErrorCode.PRODUCT_DUPLICATE_SERIAL);
  }
  try {
    productId = productRepository.insertProduct(
        ownerId, request, manufacturerNormalized, fingerprint,
        serialNumberProtector.encrypt(request.serialNumber()));                     // AES 암호화
  } catch (DataIntegrityViolationException exception) {
    throw new CommonException(ErrorCode.PRODUCT_DUPLICATE_SERIAL);   // 동시 요청은 DB 유니크 제약이 막음
  }
  ...
}
```

**조회 `get`** — 본인 제품의 암호문만 꺼내 복호화

```java
@Transactional(readOnly = true)
public ProductResponse get(long ownerId, long productId) {
  ProductResponse product = getForListing(ownerId, productId);
  String ciphertext = productRepository
      .findSerialCiphertextOwned(productId, ownerId)
      .orElseThrow(() -> new CommonException(ErrorCode.RESOURCE_NOT_FOUND));
  return product.withSerialNumber(serialNumberProtector.decrypt(ciphertext));       // AES 복호화
}
```

**수정 `update`** — 응답을 만들 때 같은 방식으로 복호화

```java
return withImages(after)
    .withSerialNumber(
        serialNumberProtector.decrypt(
            productRepository.findSerialCiphertextOwned(productId, ownerId).orElseThrow()));
```

판매글용 `getForListing`과 목록 `listMine`은 시리얼 번호를 복호화하지 않습니다.

### 4-3. DB — 마이그레이션

- `V2026101303__create_product_tables.sql`: `serial_number_fingerprint VARCHAR(64)`, `serial_number_ciphertext VARCHAR(512)` 컬럼 생성
- `V2026102001__align_product_p1_requirements.sql`: 유니크 제약을 `(manufacturer_normalized, serial_number_fingerprint)`로 변경

### 4-4. 복호화 자세히 (`SerialNumberProtector.decrypt`)

#### (1) 복호화가 일어나는 순간

내 제품 상세 조회(`get`)와 내 제품 수정 응답(`update`)에서만 일어납니다. 목록(`listMine`)과 판매글용 조회(`getForListing`)는 복호화하지 않습니다.

#### (2) 호출 전: 누구의 암호문을 꺼내는가

```java
productRepository.findSerialCiphertextOwned(productId, ownerId)
```

제품 번호(`productId`)와 **로그인한 사용자 번호(`ownerId`)가 함께** 맞는 행의 암호문만 가져옵니다. `ownerId`는 JWT에서 꺼낸 값이라 요청에 다른 사람 번호를 넣을 수 없습니다. 내 제품이 아니면 결과가 없어 `RESOURCE_NOT_FOUND`로 끝나고, **복호화 코드까지 가지 않습니다.**

#### (3) 암호문의 구조

저장된 값은 이렇게 생겼습니다.

```
Base64( [IV 12바이트] [암호화된 데이터] [인증 태그 16바이트] )
```

`110112`(6글자 = 6바이트)를 예로 들면 12 + 6 + 16 = 34바이트이고, Base64로 바꾸면 48글자입니다. 컬럼은 `VARCHAR(512)`입니다. (태그 16바이트는 `TAG_BITS = 128`에서 나옵니다. 길이 계산은 구조에서 유추한 값이며 실제로 실행해 확인하지는 않았습니다.)

#### (4) 복호화 단계별 동작

```java
byte[] combined = Base64.getDecoder().decode(ciphertext);                       // ① 글자 → 바이트
Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");                       // ② AES-GCM 도구 준비
cipher.init(Cipher.DECRYPT_MODE, encryptionKey,
    new GCMParameterSpec(TAG_BITS, combined, 0, IV_BYTES));                     // ③ 키와 IV 지정
byte[] plain = cipher.doFinal(combined, IV_BYTES, combined.length - IV_BYTES); // ④ 풀기 + 변조 검사
return new String(plain, StandardCharsets.UTF_8);                              // ⑤ 바이트 → 글자
```

| 단계 | 하는 일 | 이유 |
| --- | --- | --- |
| ① | Base64 글자를 원래 바이트로 되돌림 | DB에는 바이트를 그대로 못 넣어 글자로 바꿔 저장했기 때문 |
| ② | `AES/GCM/NoPadding` 복호화 도구를 만듦 | 암호화 때와 **같은 방식**이어야 풀림 |
| ③ | 암호화 키와 IV를 넣음. IV는 `combined`의 **앞 12바이트(0번부터 12개)** | IV는 비밀이 아니라 암호문 앞에 붙여 저장해 두었음 |
| ④ | IV 뒤(12번째)부터 끝까지가 `암호화된 데이터 + 태그`. 이것을 복호화하면서 **태그를 검사** | 태그가 맞아야 "변조되지 않았고 키도 맞다"는 뜻 |
| ⑤ | 나온 바이트를 UTF-8 글자로 바꿈 | 최종 시리얼 번호 |

#### (5) 어떤 키로 푸는가

```java
this.encryptionKey = new SecretKeySpec(derive(master, "serial-encryption"), "AES");
```

서버가 시작될 때 마스터 시크릿(`PRODUCT_SERIAL_SECRET`)에서 계산해 만든 값입니다. 같은 시크릿이면 **서버를 껐다 켜도 항상 같은 키**가 나오므로 예전에 저장한 암호문을 계속 풀 수 있습니다. 반대로 시크릿이 바뀌면 키가 달라져 풀리지 않습니다.

#### (6) 실패하면

| 상황 | 결과 |
| --- | --- |
| 암호문이 한 글자라도 바뀜 (변조) | 태그 검사 실패 → `GeneralSecurityException` → `IllegalStateException("Failed to decrypt serial number")` |
| 시크릿이 달라 키가 다름 | 태그 검사 실패 → 위와 같음 |
| Base64가 아닌 값이 들어 있음 | Base64 해석 단계에서 `IllegalArgumentException`. **`catch`에 잡히지 않고 그대로 올라감** |

GCM은 틀린 키로 풀어도 "엉뚱한 글자"가 나오지 않고 **아예 실패**합니다. 그래서 잘못된 번호가 화면에 보일 위험이 없습니다. 실패하면 서버 오류(500)가 되므로, 시크릿 변경 시 기존 데이터가 풀리지 않는 문제를 미리 점검해야 합니다. (실제 응답 코드는 `GlobalExceptionHandler` 처리에 따르며 여기서 따로 확인하지는 않았습니다.)

#### (7) 복호화 결과의 모습

암호화 전에 `normalize`를 거치므로 **복호화하면 정규화된 값**이 나옵니다. 사용자가 `ab12 cd34`로 입력했어도 `AB12CD34`로 보입니다. 입력한 원래 모양(소문자, 띄어쓰기)은 복원되지 않습니다.

#### (8) 암호화와 복호화 비교

| | 암호화 `encrypt` | 복호화 `decrypt` |
| --- | --- | --- |
| IV | 호출할 때마다 **새로 난수 생성** | 암호문 **앞 12바이트에서 읽음** |
| 입력 | 정규화된 시리얼 | 저장된 Base64 암호문 |
| 출력 | `IV + 암호문 + 태그`의 Base64 | 정규화된 시리얼 |
| 키 | 같은 암호화 키 | 같은 암호화 키 |

## 5. AES와 HMAC은 어디에 쓰이나

| | 위치 | 용도 |
| --- | --- | --- |
| AES | `SerialNumberProtector` | 시리얼 번호 암호화·복호화 (**여기에만** 사용) |
| HMAC | `SerialNumberProtector` | 시리얼 번호 지문 |
| HMAC | `SecurityConfig` | JWT 서명 (HS256 = HMAC-SHA256) |
| HMAC | `LocalFileStorage` | 임시 파일 URL 서명 |

HMAC은 "비밀 키로 값을 만들어 같은지 확인한다"는 범용 방식이라 여러 곳에서 씁니다.

### 다른 두 곳의 코드

**JWT 서명 — `common/security/SecurityConfig.java`**

```java
private static SecretKey secretKey(JwtProperties properties) {
  return new SecretKeySpec(properties.secret().getBytes(StandardCharsets.UTF_8), "HmacSHA256");
}
```

**임시 URL 서명 — `common/storage/LocalFileStorage.java`**

```java
private String sign(String key, long expires) {
  Mac mac = Mac.getInstance("HmacSHA256");
  mac.init(new SecretKeySpec(signingKey, "HmacSHA256"));
  return HexFormat.of()
      .formatHex(mac.doFinal((key + "\n" + expires).getBytes(StandardCharsets.UTF_8)));
}
```

### 키 관계 (주의)

| 용도 | 설정 | 비고 |
| --- | --- | --- |
| 시리얼 번호 | `app.product.serial-secret` (`PRODUCT_SERIAL_SECRET`) | 이 안에서 지문 키·암호화 키를 따로 파생 |
| JWT | `app.jwt.secret` | |
| 파일 URL | `app.storage.signing-secret` | `application.yaml`: `${STORAGE_SIGNING_SECRET:${app.jwt.secret}}` |

시리얼 번호 키는 JWT·파일 키와 별개입니다. 다만 **`STORAGE_SIGNING_SECRET`을 따로 설정하지 않으면 파일 URL 서명 키가 JWT 시크릿과 같은 값**이 됩니다. 운영에서는 별도 값을 설정하는 것이 좋습니다.

## 6. 보안 정리

**DB만 유출됐을 때:** 지문은 되돌릴 수 없고, 암호문은 키 없이는 풀 수 없으므로 원문을 알 수 없습니다.

**DB와 비밀 키가 함께 유출됐을 때**

- 암호문: 바로 복호화됩니다.
- 지문: 시리얼은 형식을 짐작할 수 있어서, 후보(`110000`~`119999` 등)를 키로 HMAC 계산해 지문과 대조하면 뚫릴 수 있습니다.

| 상황 | 지문 | 암호문 |
| --- | --- | --- |
| DB만 유출 | 알 수 없음 | 알 수 없음 |
| DB + 키 유출 | 후보 대입으로 뚫릴 수 있음 | 바로 복호화됨 |

그래서 키는 DB와 다른 곳(환경변수 등)에 두고 코드·로그에 남기지 않습니다. 운영 설정(`application-prod.yaml`)은 기본값 없이 `PRODUCT_SERIAL_SECRET`을 요구합니다.

## 7. 운영 주의

- 시크릿을 잃으면 기존 암호문을 복호화할 수 없습니다. 백업과 접근 통제가 필요합니다.
- 시크릿을 바꾸면 지문이 달라져 중복 검사가 깨지고 기존 암호문도 풀리지 않습니다. 교체하려면 전체 재처리 계획이 필요합니다.
- 로컬·테스트 설정의 시크릿은 개발용입니다. 운영에 쓰지 않습니다.
- 복호화는 **본인 제품에 한해서만** 됩니다. 다른 사람 제품은 `RESOURCE_NOT_FOUND`입니다.

## 8. 대안 비교

| 방법 | 장점 | 단점 |
| --- | --- | --- |
| 평문 | 단순 | DB 유출 시 노출 |
| 일반 해시만 | 중복 확인 가능 | 복원 불가, 추측 가능한 번호는 대조 공격 가능 |
| HMAC만 | 키 없이 대조 어려움 | 복원 불가 |
| 암호화만 | 복원 가능 | IV가 매번 달라 같은 번호 일치 검색 불가 |
| **HMAC + AES-GCM (채택)** | 중복 확인 + 복원 | 키 관리 필요 |

AWS KMS는 AES를 대체하는 것이 아니라 키를 안전하게 관리하는 서비스입니다(현재는 사용하지 않고 환경변수 시크릿 사용). PASETO는 로그인 토큰 형식이라 이 문제와 무관합니다.

## 9. 발표용 한 줄

"시리얼은 중복 확인용 HMAC 지문과 복원용 AES-GCM 암호문으로 나눠 저장하고, 두 키는 마스터 시크릿에서 따로 파생합니다. 최종 중복 방지는 DB 유니크 제약이 담당합니다."

테스트: `SerialNumberProtectorTest`

> 참고: `TROUBLESHOOTING.md` 4절은 "조회 흐름에서 복호화하지 않는다"고 적혀 있으나, 현재 코드는 소유자 조회·수정 응답에서 복호화합니다.

## 10. 비전공자용 10줄 정리

비유: 시리얼 번호는 **집 비밀번호**, DB는 **보관함**입니다.

1. 시리얼 번호는 노트북마다 다른 고유 번호라서, 같은 노트북이 두 번 등록되는 것을 막아야 합니다.
2. 그런데 번호를 DB에 그대로 적어 두면, DB를 해킹당했을 때 번호가 그대로 보입니다. 그래서 **원본은 저장하지 않습니다.**
3. 대신 번호로 **두 가지 변환 값**을 만들어 저장합니다. 하나는 지문, 하나는 암호문입니다.
4. **지문(HMAC)**은 번호를 믹서기에 갈아 만든 주스입니다. 같은 번호는 항상 같은 모양이 나오지만, 주스를 보고 재료(번호)를 알아낼 수는 없습니다.
5. 새 번호가 들어오면 믹서기에 갈아 보고, **이미 저장된 주스와 같은지만** 비교합니다. 같으면 "이미 등록된 노트북"이라고 막습니다.
6. **암호문(AES)**은 번호를 자물쇠 상자에 넣고 잠근 것입니다. 열쇠(비밀 키)가 있어야만 열 수 있습니다.
7. 상자는 열 때마다가 아니라 **잠글 때마다 모양이 달라집니다.** 그래서 상자끼리 비교해서는 같은 번호인지 알 수 없고, 비교는 지문이 맡습니다.
8. 주인이 자기 노트북 상세 화면을 열면, 서버가 열쇠로 상자를 열어(**복호화**) 번호를 보여 줍니다. 남의 노트북은 상자를 꺼내지도 못합니다.
9. 상자가 중간에 조금이라도 바뀌었거나 열쇠가 틀리면, 엉뚱한 번호가 나오지 않고 **아예 열리지 않습니다.**
10. 해킹으로 보관함(DB)만 털리면 주스와 잠긴 상자만 보여서 번호를 알 수 없습니다. 다만 **열쇠까지 털리면 뚫리므로**, 열쇠는 보관함과 다른 곳에 따로 보관해야 합니다.

한 문장: "지문으로 중복을 막고, 암호문으로 원본을 숨겼다가 주인에게만 다시 보여 준다."

## 11. "읽을 수 있다"의 정확한 의미

| | 읽을 수 있는 사람 | 용도 |
|---|---|---|
| HMAC 지문 | **아무도 없음** (소유자·서버·개발자 모두 원문 복원 불가, 같은 번호인지 비교만 가능) | 중복 등록 방지 |
| AES 암호문 | **서버가** 소유자에게만 풀어서 보여 줌 | 소유자의 상세 화면 표시 |

- 사용자는 키를 갖고 있지 않다. 키는 서버 설정(`PRODUCT_SERIAL_SECRET`)에만 있고, **푸는 주체는 서버**다.
- 소유자 확인은 서버 SQL에서 한다: `WHERE id = ? AND owner_id = ? AND archived_at IS NULL` (`ProductRepository.findSerialCiphertextOwned`). 다른 사용자가 요청하면 암호문이 조회되지 않아 풀 일 자체가 없다.
- 소유권이 이전되면 그다음부터는 새 소유자만 볼 수 있다.
- DB만 유출되면 원문을 알 수 없다. 키(`PRODUCT_SERIAL_SECRET`)까지 같이 유출되면 AES 암호문은 풀리므로 키는 DB와 따로 보관한다.

## 12. AES 암호문이 쓰이는 정확한 위치

암호화·복호화 클래스는 `SerialNumberProtector` 하나이고, 이를 호출하는 곳은 `ProductService`뿐이다.

| 시점 | 하는 일 | 위치 |
|---|---|---|
| 제품 등록 | 암호화해서 `serial_number_ciphertext`에 저장 (지문도 같은 INSERT) | `ProductService.create` 80줄 |
| 제품 상세 보기 | 소유자의 암호문을 복호화해서 응답 | `ProductService.get` 99~103줄 |
| 제품 수정 후 응답 | 시리얼은 수정하지 않고, 응답용으로만 복호화 | `ProductService.update` 150~153줄 |

- 등록 `create`의 마지막 줄이 `get`을 호출하므로 등록 직후 화면에서도 복호화가 한 번 일어난다.
- 컬럼은 `V2026101303__create_product_tables.sql`에서 `serial_number_ciphertext VARCHAR(512) NOT NULL`.

**암호문을 쓰지 않는 곳**
- `getForListing` (판매글 작성용): 주석에 "시리얼 조회·복호화 없이"라고 되어 있고 실제로 복호화하지 않는다. 판매글에는 시리얼 번호가 노출되지 않는다.
- `listMine` (제품 목록): 복호화하지 않는다.
- 중복 검사: 암호문이 아니라 HMAC 지문만 쓴다.
- 문서·채팅·이전·AI 등 다른 기능: `SerialNumberProtector`를 쓰지 않는다. (판매글 목록 조회 `ListingQueryRepository`는 이 클래스의 제조사 이름 정규화 함수만 가져다 쓴다.)

## 13. 코드 전체의 해시·서명·암호화 사용처

`src/main`의 Java·JS 전체에서 `MessageDigest`, `PasswordEncoder`, `BCrypt`, `HmacSHA`, `Cipher` 등을 검색한 결과다. (설정 yaml·SQL 안의 해시는 보지 않았다.)

| 방식 | 위치 | 용도 | 되돌릴 수 있나 |
|---|---|---|---|
| HMAC-SHA256 (지문) | `SerialNumberProtector` | 시리얼 중복 검사 | 불가 |
| **AES-GCM** | `SerialNumberProtector` | 시리얼 원문 복원 | 키가 있으면 가능 (**프로젝트에서 유일한 복원형**) |
| HMAC-SHA256 (서명) | `SecurityConfig` 168줄 | 로그인 토큰(JWT) 서명 | 해당 없음 |
| HMAC-SHA256 (서명) | `LocalFileStorage` 85~93줄 | 문서 임시 다운로드 URL 서명 (`isEqual`로 비교) | 해당 없음 |
| **BCrypt** | `SecurityConfig` 171~174줄, `AuthService` | **비밀번호 저장** (로그인 때 입력값을 해시해 비교) | 불가 |
| SHA-256 | `RefreshTokenStore` 60~67줄 | refresh 토큰: 원문 대신 해시를 Redis 키로 사용 | 불가 |
| SHA-256 | `OwnershipTransferService.hashToken` | 소유권 이전 토큰: 해시만 DB에 저장 | 불가 |
| SHA-256 | `DocumentService` 131, 142~145줄 | 문서 사본 교체 기록(교체된 파일의 해시만 기록) | 불가 |
| SHA-256 | `ListingSuggestionSource`, `ProductQuestionSource` | AI 입력 사실이 바뀌었는지 확인 (`fingerprint`라는 이름이지만 시리얼 보호와 무관) | 불가 |
| SHA-256 | `InternalTradeOwnershipService` 166줄 | 내부 거래 이전에서 쓰는 해시 (정확한 용도는 주변 코드를 읽지 않아 미확인) | 불가 |

- HMAC이라는 도구는 세 곳(시리얼 지문, 임시 URL 서명, JWT 서명)에서 쓰이지만 목적이 다르다. 시리얼 지문은 "같은 번호인가", 나머지 둘은 "서버가 만든 것이 맞는가"를 확인한다.
- 토큰류(이전 토큰, refresh 토큰)는 HMAC이 아니라 일반 해시다. "원문을 저장하지 않고 비교만 한다"는 점은 같다.

## 14. 왜 HMAC·해시 자리에 AES를 쓰면 안 되나

AES는 "숨겼다가 다시 꺼내 쓰는" 용도이고 HMAC·해시는 "꺼낼 일 없이 맞는지만 확인하는" 용도라서 서로 대신할 수 없다.

| 하려는 일 | 맞는 도구 | 이유 |
|---|---|---|
| 같은 값인지 비교 (중복 검사) | HMAC / 해시 | 같은 입력이면 같은 결과가 나옴 |
| 비밀번호·토큰 저장 | BCrypt / SHA-256 | 되돌릴 수 없어야 안전함 |
| 서버가 만든 것인지 검증 | HMAC | 서명 확인용 도구 |
| 나중에 원문을 읽어야 함 | AES | 키가 있으면 복원 가능 |

**시리얼 중복 검사를 AES로 못 하는 이유**
- 이 코드의 AES-GCM은 같은 번호도 암호화할 때마다 다른 값이 나온다 (무작위 IV가 앞에 붙음). 암호문끼리 비교해도 같은 번호인지 알 수 없고 DB 유니크 제약도 걸 수 없다.
- 억지로 하려면 IV를 고정해야 하는데, AES-GCM은 같은 IV를 재사용하면 안 되는 방식이라 보안이 크게 약해진다.
- 그래서 지문(HMAC)과 암호문(AES)을 **둘 다** 저장한다.

**비밀번호를 AES로 저장하면 안 되는 이유**
- AES는 키가 있으면 원문으로 풀린다. 서버 키가 유출되거나 관리자가 마음먹으면 모든 사용자의 비밀번호 원문을 볼 수 있다. 사용자가 다른 사이트에서 같은 비밀번호를 쓰면 피해가 이 서비스 밖으로 퍼진다.
- 서비스는 비밀번호 원문을 알 필요가 없다. 로그인 때 "입력한 값이 맞는지"만 확인하면 된다.
- BCrypt가 이 일에 맞는 이유: 되돌릴 수 없고, 같은 비밀번호도 저장할 때마다 값이 다르며(무작위 소금 값), 일부러 느려서 대량 추측이 어렵다.
- 시리얼의 HMAC 지문은 같은 번호면 항상 같은 값이 나와야 중복 검사가 되므로, 비밀번호 방식(BCrypt)과 일부러 다르게 쓴다.

**한 줄 정리:** 비밀번호처럼 복원할 필요가 없는 값은 BCrypt 같은 한 방향 해시로, 복원해서 다시 보여 줘야 하는 값(이 프로젝트에서는 시리얼 번호)만 AES로 저장한다.

## 15. 발표자료용 정리 (슬라이드 순서)

### 슬라이드 1. 왜 필요한가 (문제)
- 시리얼 번호는 노트북마다 다른 **고유 번호**다. 같은 기기가 **두 번 등록되면 안 된다.** (중복 등록 방지)
- 그런데 번호를 DB에 그대로 저장하면, DB가 유출될 때 **모든 시리얼 번호가 그대로 노출**된다.
- 그래서 **원문은 저장하지 않고**, 목적이 다른 두 가지 값으로 나눠 저장한다.

### 슬라이드 2. 해결 — 두 가지 값을 따로 저장

| | HMAC 지문 | AES 암호문 |
|---|---|---|
| 한마디로 | 믹서기에 간 주스 | 자물쇠 상자 |
| 사용한 이유 | **중복 등록인지 확인** | **원문 보호 + 필요할 때 복원** |
| 원문 복원 | **불가** (한 방향, 누구도 못 읽음) | 가능 (서버 키가 있을 때) |
| 같은 번호를 넣으면 | 항상 같은 값 → **비교 가능** | 매번 다른 값 → 비교 불가 |
| 저장 컬럼 | `serial_number_fingerprint` | `serial_number_ciphertext` |
| 알고리즘 | HMAC-SHA256 | AES-256-GCM |

- 원문은 어디에도 저장되지 않는다.
- 두 값은 하나의 마스터 시크릿(`PRODUCT_SERIAL_SECRET`)에서 **용도별로 따로 만든 키**를 쓴다.

### 슬라이드 3. 어떤 기능에 사용했나

| 화면·기능 | 무엇을 하나 | 사용한 것 |
|---|---|---|
| **새 제품 등록** | 시리얼 번호를 입력하면 **이미 등록된 번호인지 확인**하고, 아니면 원문 대신 지문과 암호문으로 저장 | HMAC 지문 + AES 암호화 |
| **중복 등록 방지** | 같은 제조사의 같은 시리얼이면 "이미 등록된 제품"으로 거부 (원문 없이 지문만 비교, DB 유니크 제약이 최종 방어) | HMAC 지문 |
| **내 제품 상세 보기** | 소유자가 자기 제품을 열면 저장된 암호문을 풀어서 **시리얼 번호를 화면에 표시** | AES 복호화 |
| **내 제품 정보 수정** | 수정을 저장한 뒤 화면에 시리얼을 다시 보여 주려고 복호화 (시리얼 번호 자체는 수정할 수 없음) | AES 복호화 |

**시리얼을 쓰지 않는 기능:** 내 제품 목록, 판매글 작성, 중고 거래 목록·판매글 상세, 채팅, 문서 보관함. **판매글에는 시리얼이 노출되지 않는다.**

**같은 HMAC 도구를 쓰는 다른 기능 (시리얼 보호와는 별개)**

| 기능 | 무엇을 하나 | 사용한 것 |
|---|---|---|
| **로그인** | 로그인 후 발급되는 인증 토큰이 서버가 만든 것인지 확인 (서명) | HMAC 서명 |
| **문서 보관함 → 문서 보기** | 문서를 열 때 5분짜리 임시 주소를 만들고, 주소가 위조되지 않았는지 확인 (서명) | HMAC 서명 |

**비교만 하는 다른 기능 (HMAC이 아닌 해시)**

| 기능 | 무엇을 하나 | 사용한 것 |
|---|---|---|
| **회원가입·로그인 비밀번호** | 비밀번호 원문은 저장하지 않고, 로그인할 때 입력값을 해시해서 비교 | BCrypt |
| **로그인 유지** | 로그인 유지용 토큰의 원문은 저장하지 않고 해시만 보관 | SHA-256 |
| **소유권 이전 링크** | 이전 링크의 토큰 원문은 저장하지 않고 해시만 DB에 보관 | SHA-256 |

- **AES 암호화는 시리얼 번호에서만** 쓴다. 다시 보여 줘야 하는 민감 정보는 시리얼 번호뿐이기 때문이다.

### 슬라이드 4. 보안 효과
- **DB만 유출돼도** 시리얼 원문을 알 수 없다 (지문과 암호문만 보인다).
- **소유자에게만 복호화한다.** 조회 SQL에 `owner_id` 조건이 있어, 다른 사용자가 요청하면 암호문 자체가 조회되지 않는다. 소유권이 이전되면 새 소유자만 볼 수 있다.
- **변조 감지.** AES-GCM은 암호문이 조금이라도 바뀌면 복호화에 실패한다 (엉뚱한 값이 나오지 않는다).
- **같은 번호도 암호문은 매번 달라** 암호문 모양으로 같은 번호를 추측할 수 없다 (무작위 IV).
- 주의: **서버 키까지 함께 유출되면** 암호문은 풀린다. 그래서 키는 DB와 분리해서 보관한다.

### 슬라이드 5. 왜 두 개를 모두 쓰나 (예상 질문)

**Q. AES만 쓰면 안 되나요?**
A. AES 암호문은 같은 번호도 매번 다르게 나와서 **중복 검사가 안 됩니다.** 중복 검사는 같은 번호가 같은 값이 되는 HMAC이 맞고, 원문을 다시 보여 주는 것은 AES가 합니다.

**Q. HMAC만 쓰면 안 되나요?**
A. HMAC은 한 방향이라 **소유자에게 시리얼을 다시 보여 줄 수 없습니다.**

**Q. 비밀번호도 같은 방식으로 저장하나요?**
A. 아니요. 비밀번호는 **BCrypt**(한 방향, 복원 불가)로 저장합니다. 복원해서 보여 줄 필요가 없는 값은 복원 가능하게 저장하지 않는 것이 원칙입니다. **복원 가능한 형태로 저장하는 민감 정보는 시리얼 번호뿐입니다.**

**Q. HMAC은 시리얼 말고 다른 곳에도 쓰나요?**
A. 로그인 토큰(JWT) 서명과 문서 임시 다운로드 URL 서명에도 쓰입니다. 다만 목적이 다릅니다. 시리얼 지문은 "같은 번호인가", 나머지는 "서버가 만든 것이 맞는가"를 확인합니다.

**Q. 키가 유출되면요?**
A. 암호문은 풀릴 수 있습니다. 그래서 키(`PRODUCT_SERIAL_SECRET`)는 DB와 분리하고, 운영 환경에서는 환경 변수로만 주입합니다. 지문(HMAC)은 키가 유출돼도 원문으로 되돌릴 수는 없지만, 키가 있으면 후보 번호를 대입해 맞는지 확인할 수는 있습니다.

### 슬라이드 6. 한 줄 요약
> "시리얼 번호는 원문 대신 **중복 확인용 HMAC 지문**과 **복원용 AES-GCM 암호문**으로 나눠 저장한다. 지문으로 중복 등록을 막고, 암호문으로 원문을 숨겼다가 **소유자에게만** 서버가 풀어서 보여 준다."

### 슬라이드에 넣을 한 장짜리 도식 (텍스트)

```
시리얼 번호 입력  "110112"
        │
        ├─▶ HMAC-SHA256 ──▶ 지문   ──▶ DB 저장 + 중복 검사 (원문 복원 불가)
        │
        └─▶ AES-256-GCM ──▶ 암호문 ──▶ DB 저장
                                         │
              소유자가 상세 화면을 열면 ◀─┘  서버가 복호화해서 보여 줌
                                           (다른 사용자는 조회 자체가 안 됨)
```
