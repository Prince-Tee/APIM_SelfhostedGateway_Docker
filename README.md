# Self-Hosted Gateway with Azure API Management using Docker and SSL

## 🚧 Prerequisites

* An **Azure subscription**
* **Docker** installed on a virtual machine (Linux or Windows)
* A **domain name** (acquired via **Azure App Service Domains**)
* An **SSL certificate** (generated from [ZeroSSL](https://zerossl.com))

---

## ✅ Step-by-Step Implementation

### 1. Provision Azure API Management (Developer Tier)

* In the **Azure Portal**, create a new **API Management (APIM)** instance using the **Developer SKU**—required for self-hosted gateway setup.
* Wait for the deployment to complete.

> *📸 Insert screenshot here*

---

### 2. Provision the Virtual Machine

* Created a Virtual Machine (VM) on Azure.

> *📸 Insert screenshot here*

* For testing, opened required ports to allow traffic from any source (⚠️ Not recommended for production).

> *📸 Insert screenshot here*

---

### 3. Purchase and Configure Custom Domain

* Purchased the domain `taiwoai2.com` from **Azure App Service Domains**.
* Added a **DNS A Record** to point the domain to the VM’s **public IP address**:
  *Azure Portal > Manage DNS records > Record sets > Add A record*

> *📸 Insert screenshots here*

---

### 4. Obtain and Prepare SSL Certificate

* Generated a free SSL certificate via **ZeroSSL**.
* Verified domain ownership by creating a required CNAME record.
* Downloaded the certificate bundle and converted it to `.pfx` format using OpenSSL:

```bash
openssl pkcs12 -export \
  -out taiwoai2.pfx \
  -inkey private.key \
  -in fullchain.crt \
  -name taiwoai2
```

> *📸 Insert screenshots here*

---

### 5. Create a Self-Hosted Gateway

* In **APIM > Gateways**, click **+ Add**, provide a name (e.g., `taiwo`), and associate the gateway with desired APIs.

> *📸 Insert screenshots here*

---

### 6. Configure Custom Hostname in APIM

* Go to **APIM > Custom Domains** and add your domain under the **Gateway Endpoint** section.
* Upload the `.pfx` certificate and bind it to the custom hostname.

> *📸 Insert screenshots here*

* Navigate to your API > Settings > Gateway, and switch from "Managed" to the newly created gateway.

---

### 7. Upload SSL Certificate to Azure

* Uploaded the `.pfx` file under **APIM > Certificates**.
* Associated the uploaded certificate with the custom domain in **Custom Domains** > Gateway section.

---

### 8. Configure Gateway Settings & Docker Environment

* Created an `env.conf` file with required configuration values.
* Assumed Docker is already installed and set up on the VM.

---

### 9. Run the Self-Hosted Gateway Container

```bash
docker run -d \
  --name taiwo \
  -v ~/downloads/taiwoai2_cert/taiwoai2.pfx:/taiwoai2.pfx \
  -v ~/downloads/taiwoai2_cert/env.conf:/env.conf \
  -p 80:8080 -p 443:8081 \
  mcr.microsoft.com/azure-api-management/gateway:v2
```

> *📸 Insert screenshot of running container*

---

### 🔐 Gateway Certificate Authority Configuration (REST API)

Due to limitations with certificates in self-hosted gateways (as noted [here](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-ca-certificates)), you must **associate the certificate via REST API**:

**Endpoint:**

```
PUT https://management.azure.com/subscriptions/{subId}/resourceGroups/{rg}/providers/Microsoft.ApiManagement/service/{apimName}/gateway/certificate-authority/{certificateId}?api-version=2022-08-01
```

**Body:**

```json
{
  "properties": {
    "certificate": {
      "id": "/subscriptions/.../certificates/xxx"
    },
    "isTrusted": true
  }
}
```

> ✅ `isTrusted = true`: For root certificates
> ⚠️ Improper configuration can prevent certificate validation inside Docker runtime.

**Validation:**

```bash
docker logs taiwo | grep CertificateAddedToStore
```

---

### 🧪 Testing and Validation

* Verified APIs via Postman using HTTP, HTTPS, the VM’s Public IP, and the domain name.

> *📸 Insert Postman screenshots here*

* Confirmed certificate installation and container logs:

```bash
docker logs -f taiwo | grep CertificateAddedToStore
```

* Debugged connectivity by curling the domain from inside the Docker container:

```bash
curl -vk https://taiwoai2.com
```

> *📸 Insert screenshot of `curl` test*

---

## ❌ Errors Encountered & ✅ Resolutions

| Error                                                      | Cause                               | Solution                                                |
| ---------------------------------------------------------- | ----------------------------------- | ------------------------------------------------------- |
| `openssl: command not found`                               | OpenSSL not installed               | Installed OpenSSL and updated system path               |
| Self-signed certificate showing (`test.apim.net`)          | SNI mismatch; raw IP used           | Used domain name (`https://taiwoai2.com`) instead of IP |
| Docker container crash (`config.service.endpoint` missing) | Missing or incorrect `env.conf`     | Fixed formatting and rechecked content                  |
| `404 Resource Not Found`                                   | API not correctly routed            | Published API and tested with correct path/protocol     |
| `Unable to verify first certificate`                       | Incomplete cert chain               | Used fullchain.crt during `.pfx` conversion             |
| Multiple containers conflicting                            | Confusion from duplicate containers | Ran `docker ps`, stopped and removed duplicates         |

```bash
docker ps
docker stop <container_id>
docker rm <container_id>
```

---
## Key Concepts
SNI (Server Name Indication) requires the client to use a hostname, not an IP address, for proper SSL handshake.

The SSL certificate will only respond when the request matches the bound hostname (e.g., taiwoai2.com).

The gateway token is valid for 30 days from creation — even if Azure shows a dynamic date.

## 📅 Follow-Up Notes
We confirmed with logs and behavior that:

SSL is used only when domain matches the bound hostname.

Gateway creation date can be verified in Activity Logs under Microsoft.ApiManagement/service/gateways/write.


## ✅ Final Validation Summary

* Verified SSL certificate registration via Docker logs
* Confirmed API routing works through custom domain and VM public IP
* Ensured REST API trust configuration for gateway certificates
* Successfully tested with Postman and `curl` from inside container


Feel free to reach out to me if you need clarification or assistance. Thank you.