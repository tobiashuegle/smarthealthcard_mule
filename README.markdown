**Overview**

SMART Health Cards are portable paper or digital versions of clinical information, such as vaccination history or test results. They allow consumers to keep a copy of a medical record on hand and easily share this information with others in an interoperable format when required. The initiative is part of the SMART Health IT program, an open standards and technologies initiative supported by all major healthcare IT and technology companies and now a requirement for interoperability as part of the 21st Century Cures Act in the US.

This document describes the basic concept of SMART Health IT cards and how to implement them using the MuleSoft Anypoint platform. It will provide a sample implementation and document how to adapt it to accelerate SMART Health Card implementations with MuleSoft’s AnyPoint Platform.

**How SMART Health Cards work**

Government agencies and healthcare organizations can issue SMART Health Cards to their clients in much the same way they would previously have issued paper certificates for vaccination or test results. However, SMART Health Cards are much more portable as they can be added to digital wallets and for example submitted during a flight booking through APIs. They are also much more secure as they use state of the art digital signatures based on similar cryptographic techniques used in Bitcoin.

**Creating SMART Health Cards with MuleSoft’s AnyPoint Platform**

The SMART Health team provides excellent documentation of the standards framework as well as a very good developer portal to assist with testing a standards compliant implementation at each step of the workflow.

In summary the process of creating a SMART Health Card in MuleSoft consists of the following steps:

(1) Create the Payload in the SMART Health Card FHIR format
(2) Minimise the size: minify & compress
(3) Digitally sign and create a JWS (JSON Web Signature)
(4) Issue SMART Health Card through FHIR API

**Step 1: Assembling the SMART Health Card Payload**

The first step is to integrate with the systems of record that carry the patient demographics, vaccination or test result data and expose this data through a System API. This could be an EMR system or other database such as an Immunization Register. The  Accelerator for Healthcare provides valuables assets to accelerate this integration for example with its FHIR System APIs for Cerner and Epic.

The resulting FHIR bundle which consists of the patient demographics and their vaccination records is inserted into a Verifiable Credential (VC) structure. MuleSoft’s DataWeave and message transformation functions make it very easy for developers to convert data from the format and structure as received of the source system into the desired SMART Health Card format using drag & drop.

**Step 2: Minimise the size: minify & compress**

To ensure the SMART Health Card can be encoded in a single QR code the payload needs to be as small as possible. Therefore the first step is to remove all white space (minify) the payload. Again MuleSoft makes this very easy by simply adding indent=false to the output in a transform processor.


output application/json indent=false

Next the payload needs to be compressed using the DEFLATE algorithm omitting any headers. While MuleSoft provides a compression processor it unfortunately adds the header making it unsuitable for the SMART Health Card format. We therefore have implemented this in a very basic Java Class that converts the minified JSON string into a byte array and compresses it.

payloadCompressed = DeflateUtils.compress(byteArray);


**Step 3: Digitally sign and create a JWS (JSON Web Signature)**

To ensure the authenticity of the SMART Health Card they are digitally signed using public/private key encryption. The first step is to define the public & private keys in the form of a JWK (JSON Web Key). The SMART Health Card specification requires the use of the P-256 Elliptic Curve with algorithm ES256. The key id is the base64url SHA-256 thumbprint of the key. During development you may use https://mkjwk.org/ where you can easily create these keys. 

//Sample JSON Web Key (Private Key)
{
  "kty": "EC",
  "kid": "3Kfdg-XwP-7gXyywtUfUADwBumDOPKMQx-iELL11W9s",
  "use": "sig",
  "alg": "ES256",
  "crv": "P-256",
  "x": "11XvRWy1I2S0EyJlyf_bWfw_TQ5CJJNLw78bHXNxcgw",
  "y": "eZXwxvO1hvCY0KucrPfKo7yAyMT6Ajc3N7OkAB6VYy8",
  "d": "FvOOk6hMixJ2o9zt4PCfan_UW7i4aOEnzj76ZaCI9Og"
}


Add your private key to the Mule flow and your public key, which is the same JWK without the “d” parameter, to the publicly accessible URL specified in the SMART Health Card payload.

To create a SMART Health Card compliant signature requires the use of a Java class which you can re-use from the sample app provided on Github. The implementation is again very simple requiring only a few lines of code to create a JWS (JSON Web Signature) which consists of a concatenated string with three elements:

JWSHeader.Payload.Signature
JWS Header = {"zip":"DEF","alg":"ES256","kid": base64url-encoded SHA-256 JWK Thumbprint}
Payload = base64url encoded compressed payload
Signature = base64url encoded signature 

**Step 4: Issue the SMART Health Card through a FHIR API**

The SMART Health card is formed by wrapping the JSON Web Signature into a specific verifiable Credential format. The following shows an example of a digitally signed SMART Health Card for COVID vaccinations.

{"verifiableCredential": [     "eyJ6aXAiOiJERUYiLCJhbGciOiJFUzI1NiIsImtpZCI6IjNLZmRnLVh3UC03Z1h5eXd0VWZVQUR3QnVtRE9QS01ReC1pRUxMMTFXOXMifQ.7VRNj9owEL3zKyL3CvmA3YXl2K1U7WGrSkv3suJgnIG4cuzIdhAU5b937MBuIEFQtadqIyWyx29mnmfmZdcLAsKNIdOAZNYWZhpFpgAWmpxqmwEVNgsZ1amJYEPzQoCJEF6CJn3nKhdLdE3uRjfjye1kOA5vR3f-YM3QvsMVru22ANy9-l3QSHSa41O9GbiNj38BzvO8lPwXtVzJa_BMrXma3BOPnNcOhGlIQVpOxXO5-AnMvhHHw2XG9Qto4xJgiW7COEzeM7nTz6VMBTR80K7BqFIzmNUXJ3tM_x2xLwlhSghM2eSPp0hHbxsVc8-usXapSyF-aOFiHLJN40aMIx5H7M5w_I5VxLwnIVyLaQ4nXLoYHXjRnAvHnTyVAp7Vsh3R41Z8DbIzrD9-ohuOgTiVne6ImGUQPEoLK02t0qQDNG_ZqhPLvHXZBceR-UKtL0lyf5sM4mQwjI_DN8NU_T_rUfI3PXrsmvc3tLHUlqaeKqdUC2kbtKaMcQkPKu3K6OSgUi5XZzqz6-6F2RoL-eEfgtLLxDhUehU5gUSGpxFbb841ktVUyDCedHWx6l3qa9W6ZLGf5c4LaliCBukr3tTOxaiKsVJ7TzcgM57vaQ_9kMRJu9gF6KXSOf4sr9YPZW6ap-cqnXJTCOr19RVAKLkKXnxH_UwED4JLzq4r42UxCGW_lfnC0ycxPkmC7z8Uw_BDDP-jGEYfYjhaH3pUW9y36lW_AQ.KEaI-bjzjOYU6khYEAlWUPVO2Y0TuJgLEPcAMLVuDa_6qHraYiY5L7PzThWGGZFOd_EMg_2sr0bGnMZj0ovbcg" ]} 


The FHIR Implementation Guide for SMART Health Cards proposes a FHIR endpoint that allows applications such as a digital wallet to request a SMART Health Card. In MuleSoft this can be easily implemented as an Experience API which returns a SMART Health Card Verifiable Credential as shown above. Authentication and Authorisation are outside the scope of the SMART Health Card specification. However, they are designed to work well with the SMART on FHIR for which MuleSoft also provides an implementation template as part of our Accelerator for Healthcare. 

Alternatively the SMART Health card can be encoded as QR Code and issued to a client so they can print it or add to their digital wallet. Performing the above step in reverse allows a verifier application to decompress and display the payload containing the client identification and vaccination details as well as confirming the digital signature against the public key to ensure its authenticity. 

When combined with a trust framework such as the The Commons Project this allows for SMART Health Cards to be both verified against its authenticity and them being issued by a trusted issuer such as a government agency or health service. 



