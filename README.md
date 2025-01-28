# Kubernetes User Certificate Issuance and Authorization

This guide outlines the steps to create a user certificate, issue a certificate signing request (CSR), and provide authorization to access resources in Kubernetes.

## 1. Authentication

### 1.1 User End: Generate Private Key and CSR

1. **Generate Private Key for the user (e.g., John)**

    ```bash
    openssl genrsa -out John.key 2048
    openssl req -new -key John.key -out John.csr -subj "/CN=John"
    ```

    This will create the **John.key** (private key) and **John.csr** (certificate signing request).

### 1.2 Create a Certificate Signing Request (CSR) in Kubernetes

1. Create a YAML file for the `CertificateSigningRequest` manifest:

    ```yaml
    apiVersion: certificates.k8s.io/v1
    kind: CertificateSigningRequest
    metadata:
      name: myuser  # Replace with the username
    spec:
      request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBNQTBHQTFVRUF3d0dZVzVuWld4aE1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJDZ0tDQVFFQTByczhJTHRHdTYxakx2dHhWTTJSVlRWMDNHWlJTWWw0dWluVWo4RElaWjBOCnR2MUZtRVFSd3VoaUZsOFEzcWl0Qm0wMUFSMkNJVXBGd2ZzSjZ4MXF3ckJzVkhZbGlBNVhwRVpZM3ExcGswSDQKM3Z3aGJlK1o2MVNrVHF5SVBYUUwrTWM5T1Nsbm0xb0R2N0NtSkZNMUlMRVI3QTVGZnZKOEdFRjJ6dHBoaUlFMwpub1dtdHNZb3JuT2wzc2lHQ2ZGZzR4Zmd4eW8ybmlneFNVekl1bXNnVm9PM2ttT0x1RVF6cXpkakJ3TFJXbWlECklmMXBMWnoyalVnald4UkhCM1gyWnVVV1d1T09PZnpXM01LaE8ybHEvZi9DdS8wYk83c0x0MCt3U2ZMSU91TFcKcW90blZtRmxMMytqTy82WDNDKzBERHk5aUtwbXJjVDBnWGZLemE1dHJRSURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBR05WdmVIOGR4ZzNvK21VeVRkbmFjVmQ1N24zSkExdnZEU1JWREkyQTZ1eXN3ZFp1L1BVCkkwZXpZWFV0RVNnSk1IRmQycVVNMjNuNVJsSXJ3R0xuUXFISUh5VStWWHhsdnZsRnpNOVpEWllSTmU3QlJvYXgKQVlEdUI5STZXT3FYbkFvczFqRmxNUG5NbFpqdU5kSGxpT1BjTU1oNndLaTZzZFhpVStHYTJ2RUVLY01jSVUyRgpvU2djUWdMYTk0aEpacGk3ZnNMdm1OQUxoT045UHdNMGM1dVJVejV4T0dGMUtCbWRSeEgvbUNOS2JKYjFRQm1HCkkwYitEUEdaTktXTU0xMzhIQXdoV0tkNjVoVHdYOWl4V3ZHMkh4TG1WQzg0L1BHT0tWQW9FNkpsYWFHdTlQVmkKdjlOSjVaZlZrcXdCd0hKbzZXdk9xVlA3SVFjZmg3d0drWm89Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
      signerName: kubernetes.io/kube-apiserver-client
      expirationSeconds: 86400  # 1 day
      usages:
        - client auth
    ```

2. Apply the YAML file to create the CSR:

    ```bash
    kubectl apply -f request.yml
    ```

### 1.3 Admin Approval

1. As an admin, approve the CSR request:

    ```bash
    kubectl certificate approve John
    ```

2. To check the status of the CSR:

    ```bash
    kubectl get csr
    ```

3. Export the issued certificate:

    ```bash
    kubectl get csr myuser -o jsonpath='{.status.certificate}' | base64 -d > myuser.crt
    ```

### 1.4 Configuring kubectl to Use the New User

1. Set the credentials for the user (John):

    ```bash
    kubectl config set-credentials John --client-certificate=John.crt --client-key=John.key --embed-certs=true
    ```

2. Create a new context for the user:

    ```bash
    kubectl config set-context John --cluster=<cluster-name> --user=John --namespace=default
    ```

3. Switch to the newly created context:

    ```bash
    kubectl config use-context John
    ```

4. Verify the current user:

    ```bash
    kubectl auth whoami
    ```

5. check if John can list the pods

    ```bash
    kubectl auth can-i get pods --as John
    ```
      you get `NO` as output
---

## 2. Authorization

Kubernetes supports several methods of user authorization:

- **ABAC** (Attribute-Based Access Control)
- **RBAC** (Role-Based Access Control) – Most commonly used
- **Node-based Authorization**
- **Webhook-based Authorization** (Open Policy Agent)

### 2.1 RBAC: Role and RoleBinding Example

To give John access to read pods, create a Role and RoleBinding:

1. **Create a Role** (`pod-reader`):

    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      namespace: default
      name: pod-reader
    rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["get", "watch", "list"]
    ```

2. **Create a RoleBinding**:

    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: read-pods
      namespace: default
    subjects:
    - kind: User
      name: John
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: Role
      name: pod-reader
      apiGroup: rbac.authorization.k8s.io
    ```

3. Apply the YAML files:

    ```bash
    kubectl apply -f podaccessrole.yaml
    kubectl apply -f podaccessrolebinding.yaml
    ```

### 2.2 Verifying Access

1. Check if John has access to read pods:

    ```bash
    kubectl auth can-i get pods --as John
    ```

---

## 3. Conclusion

- **Authentication**: Issue certificates for users and approve them using Kubernetes Certificate Signing Requests (CSR).
- **Authorization**: Use RBAC to grant roles and permissions, and ensure the right access level for users.

---

