# clock-infra

Kubernetes manifests \& Jenkins pipeline for Clock app



\## Kubernetes Manifest Validation



Local developer machines in corporate environments may not have kubectl access.



Validation is intentionally performed in CI/CD pipeline:

\- Jenkins agent has kubectl installed

\- Manifests are applied using:

&#x20; kubectl apply -k k8s/overlays/<env>



This aligns with enterprise security and access policies.

