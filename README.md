Voici la **transformation complète de ton prompt en un fichier `README.md` propre, structuré et directement exploitable**.
Le fond technique est **strictement identique**, seule la forme est adaptée au standard documentation.

---

# 📱 Mobile App Send Money Module

**Digipartner – Mobile App Adapter (MTN MoMo)**

## 📌 Contexte & Principe Fondamental

> 🔴 **L’inbound Digipartner fourni EST LA RÉFÉRENCE ABSOLUE**
> 🔴 **Le module Mobile App doit être codé EXACTEMENT avec la même philosophie, la même logique et la même structure**
> 👉 Le Mobile App agit uniquement comme **un adaptateur propre** :
>
> * Payload Mobile App
> * Mapping strict vers l’inbound de référence
> * Exécution réelle via **MTN MoMo Connector**

⚠️ **Aucune improvisation, aucune duplication, aucun contournement**.

---

## 🎯 Objectif Technique Final

Construire un **module Mobile App – Send Money** qui :

* reprend **la même architecture que `inbound`**
* expose des endpoints dédiés Mobile App
* mappe strictement vers `TransactionRequest`
* utilise **TransactionService / RouterAsync / ConnectorFactory**
* appelle le **connector MTN MoMo**
* supporte :

  * initiate transfer
  * check status
  * callback
* est **testable immédiatement** avec des paramètres réels

---

## 🧱 Architecture du Module Mobile App

👉 **Même logique que `inbound`**, sans impacter l’existant.

```text
com.digipay.app.digipartner.main.rest.v1.mobileapp
├── api
│   └── MobileTransferResource.java
│
├── dto
│   ├── MobileTransferRequest.java
│   ├── MobileTransferResponse.java
│   ├── RecipientData.java
│   └── SourceWallet.java
│
├── mapper
│   └── MobileToInboundMapper.java
│
├── service
│   └── MobileTransferService.java
│
├── util
│   └── MobileValidationService.java
│
└── MobileTransactionStatus.java
```

### 📌 Règles d’architecture

* `TransactionService`, `RouterAsync`, `ConnectorFactory`, `ValidationService` **restent inchangés**
* Le module Mobile App est **un adaptateur**, pas une nouvelle logique métier

---

## 📦 DTO Mobile App (Payload Officiel)

### `MobileTransferRequest.java`

```java
@Data
public class MobileTransferRequest {

    private String transferType;              // mobile_money
    private String destinationCountry;
    private double amount;
    private String localCurrency;
    private String destinationCurrency;

    private RecipientData recipientData;
    private SourceWallet sourceWallet;
}
```

### `RecipientData.java`

```java
@Data
public class RecipientData {
    private String name;
    private String reason;
    private String country;
    private String countryCode;
    private String operator;
    private String operatorId;
    private String phoneNumber;
}
```

### `SourceWallet.java`

```java
@Data
public class SourceWallet {
    private String partnerId;
    private String partnerName;
    private String phoneNumber;
    private String countryCode;
}
```

---

## 🔁 Mapper Critique : Mobile → Inbound

### `MobileToInboundMapper.java`

```java
public class MobileToInboundMapper {

    public static TransactionRequest map(MobileTransferRequest mobile) {

        String[] names = mobile.getRecipientData().getName().split(" ", 2);

        return TransactionRequest.builder()
                .intent("inc_to_wallet")
                .transactionCreationTime(System.currentTimeMillis())

                // DESTINATION
                .destinationWalletName("MoMo")
                .beneficiaryPhoneNumber(
                        mobile.getRecipientData().getCountryCode().replace("+", "") +
                        mobile.getRecipientData().getPhoneNumber()
                )
                .beneficiaryFirstName(names[0])
                .beneficiaryLastName(names.length > 1 ? names[1] : "NA")
                .beneficiaryCountry("CG")
                .beneficiaryCurrency("XAF")

                // SENDER
                .senderMobilePhone(
                        mobile.getSourceWallet().getCountryCode().replace("+", "") +
                        mobile.getSourceWallet().getPhoneNumber()
                )
                .senderCountry("CI")
                .senderCurrency(mobile.getLocalCurrency())

                // KYC MINIMUM
                .beneficiaryBirthdate("01/01/1990")
                .beneficiaryAddress("Brazzaville")
                .beneficiaryIdType("NATIONAL_ID")
                .beneficiaryIdNumber("CG123456")

                .senderBirthdate("01/01/1985")
                .senderAddress("Abidjan")
                .senderIdType("NATIONAL_ID")
                .senderIdNumber("CI998877")

                // AMOUNTS
                .senderAmount(BigDecimal.valueOf(mobile.getAmount()))
                .beneficiaryAmount(BigDecimal.valueOf(mobile.getAmount()))

                // META
                .issuertrxref(UUID.randomUUID().toString())
                .requestType("credit")
                .purpose(mobile.getRecipientData().getReason())
                .fundOrigin("MobileApp")
                .transactionReason("Mobile App Transfer")
                .amlCheck("yes")

                .build();
    }
}
```

✅ Toutes les règles du `ValidationService` sont respectées
✅ Aucun hack, aucune bidouille

---

## ⚙️ Service Mobile App (Orchestration)

### `MobileTransferService.java`

```java
@Stateless
public class MobileTransferService {

    @Inject
    TransactionService transactionService;

    public Response initiate(
            MobileTransferRequest mobileRequest,
            String digipayToken,
            String originatingCountry,
            String callbackUrl
    ) {
        return transactionService.processInitialTransactionAndRoute(
                MobileToInboundMapper.map(mobileRequest),
                digipayToken,
                originatingCountry,
                callbackUrl
        );
    }
}
```

---

## 🌐 API REST Mobile App

### `MobileTransferResource.java`

```java
@Path("/mobileapp/api/v1/transfers")
@Consumes(MediaType.APPLICATION_JSON)
@Produces(MediaType.APPLICATION_JSON)
@Stateless
public class MobileTransferResource {

    @Inject
    MobileTransferService mobileTransferService;

    @POST
    @Path("/initiate")
    public Response initiate(
            @HeaderParam("digipay-access-token") String digipayToken,
            @HeaderParam("originating-country-id") String originatingCountry,
            @HeaderParam("callbackUrl") String callbackUrl,
            MobileTransferRequest request
    ) {
        return mobileTransferService.initiate(
                request,
                digipayToken,
                originatingCountry,
                callbackUrl
        );
    }
}
```

---

## 🧪 Test Réel – MTN MoMo

### Headers (Postman)

```http
digipay-access-token: eyJhbGciOiJIUzI1NiJ9...
originating-country-id: CI
callbackUrl: http://localhost:2020/mobileapp/callback
```

### Payload

```json
{
  "transferType": "mobile_money",
  "destinationCountry": "Congo",
  "amount": 10000,
  "localCurrency": "XOF",
  "destinationCurrency": "XAF",
  "recipientData": {
    "name": "Marie Nguema",
    "reason": "Gift",
    "country": "Congo",
    "countryCode": "+242",
    "operator": "MTN Momo",
    "operatorId": "mtn_cg",
    "phoneNumber": "069091926"
  },
  "sourceWallet": {
    "partnerId": "mtn_money",
    "partnerName": "MTN Momo",
    "phoneNumber": "0547891234",
    "countryCode": "+225"
  }
}
```

---

## ✅ Résultat Attendu

```json
{
  "success": true,
  "transferId": "TRF1702547289123",
  "status": "pending",
  "message": "Transfer initiated successfully",
  "ussdPrompt": "Please dial *126# to authorize the payment"
}
```

---

## 🧠 Conclusion

✔ Architecture **100 % alignée inbound Digipartner**
✔ Aucun contournement
✔ Connector **MTN MoMo réellement invoqué**
✔ Mobile App = **adaptateur propre et maîtrisé**

---

## 🚀 Prochaines Étapes

Choisir la suite :

1. Implémenter le **mock MTN MoMo Connector**
2. Ajouter `GET /mobileapp/api/v1/transfers/{id}/status`
3. Générer un **diagramme de séquence (code-level)**
4. Ajouter la **gestion des fees & taux de change**

👉 **Donne le numéro et on continue immédiatement.**
