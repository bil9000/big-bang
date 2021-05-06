# Prerequisites:
* How the Prerequisites docs are organized:
  * This README.md is meant to be a high level overview of prerequsites.
  * /docs/guides/prerequsites/(some_topic).md files are meant to offer more specific guidance on prerequisites while staying generic. 
  * /docs/guides/deployment_scenarios/(some_topic).md files may also offer additional details on prerequesites specific to the scenario. 
* Prerequisites vary depending on deployment scenario, thus a table is used to give an overview.
* Note for future edits: The following table was generated using tablesgenerator.com/markdown_tables, recommended to copy the table's raw text contents, visit tablesgenerator.com/markdown_tables, and File -> Paste table data when edits are needed.

| Prerequisites(rows) vs Deployment Scenarios(columns)                                                                                                                                                                                              | QuickStart                                                                                                                                                                                                                                                                                                   | Internet Connected                                                                                                                                                                                                         | Internet Disconnected                                                                                                                                                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **[OS Preconfigured](os_preconfiguration.md) and Prehardened** <br>(OS and level of hardening required depends on AO)                                                                                                                             | Prerequisite <br>Recommended: A non-hardened single VM with 8 cores and 64 GB ram <br>Minimum: 4 cores and 16 GB ram (requires overriding helm values)                                                                                                                                                       | Prerequisite <br>(CSPs usually have marketplaces with pre-hardened VM images)                                                                                                                                              | Prerequisite <br>(configured to AO's risk tolerance / mission needs)                                                                                                                                |
| **[Kubernetes Distribution Preconfigured](kubernetes_preconfiguration.md) to Best Practices and Prehardened** <br>(Any CNCF Kubernetes Distribution will work as long as an AO signs off on it)                                                   | k3d is recommended for demos (It's quick to set up, ships with a dockerized LB, works on every cloud, and bare metal)                                                                                                                                                                                        | Prerequisite <br>(https://repo1.dso.mil/platform-one/distros)                                                                                                                                                              | Prerequisite <br>(users are responsible for airgap image import of container images needed by chosen Kubernetes Distribution)                                                                       |
| **Default Storage Class** <br>((for Dynamic PVCs), the SC needs to support RWX (Read Write Many) Access Mode to support HA deployment of all BigBang AddOns)                                                                                      | Presatisfied* <br>(*if using k3d, which has dynamic local volume storage class baked in)                                                                                                                                                                                                                     | Prerequisite <br>It's recommended that users start with a CSP specific or Kubernetes Distro provided storage class                                                                                                         | Prerequisite <br>[BB Team's research spike comparing Cloud Agnostic Storage Solutions](https://repo1.dso.mil/platform-one/big-bang/bigbang/-/issues/260)                                            |
| **Support for Automated Provisioning of Service Type Load Balancer** <br>(is recommended)                                                                                                                                                         | Presatisfied* <br>(*if using k3d, which ships with the ability to add flags to treat the VM's port 443 as Kubernetes Service of Type LB's port 443, automation in the quick start repo leverages these flags)                                                                                                | Prerequisite <br>Kubernetes Distributions usually have CSPs specific flags you can pass to the kube-apiserver to support auto provisioning of CSP LBs.                                                                     | Prerequisite <br>[(See docs for guidance on bare metal and no IAM scenarios)](kubernetes_preconfiguration.md#service-of-type-load-balancer)                                                         |
| **Access to Container Images** <br>(IronBank Image Pull Credentials or AirGap import from .tar.gz's)                                                                                                                                              | Prerequisite <br>(Anyone can go to login.dso.mil, and self register against P1's SSO. That can be used to login to registry1.dso.mil to generate image pull credentials for the QuickStart)                                                                                                                  | BigBang customers are recommended to use ask their BB Customer Liaison's for an IronBank Image pull robot account, which lasts 6 months.                                                                                   | Prerequisite <br>(Airgap import of container images, [BigBang Releases](https://repo1.dso.mil/platform-one/big-bang/bigbang/-/releases) includes a .tar.gz of IronBank Images)                      |
| **Customer Controlled Private Git Repo** <br>(for GitOps, the Cluster needs network access & Credentials for Read Access to said Git Repo)                                                                                                        | Presatisfied <br>(the turn key demo points to public repo1, but you won't be able to customize it)                                                                                                                                                                                                           | Prerequisite <br>(or follow Air gap docs)                                                                                                                                                                                  | Prerequisite <br>(Air gap docs assist with provisioning an ssh based git repo)                                                                                                                      |
| **Encrypt Secrets as Code** <br>(Use SOPS + CSP KMS or PGP to encrypt secrets that need to be stored in the GitRepo)                                                                                                                              | Presatisfied <br>(Demo Repo has mock secrets encrypted with a demo PGP public encryption key)                                                                                                                                                                                                                | Prerequisite <br>(CSP KMS and IAM is more secure that gpg key pair)                                                                                                                                                        | Prerequisite <br>(Use CSP KMS if available, PGP works universally, [Flux requires the private PGP key to not have a passphrase](https://toolkit.fluxcd.io/guides/mozilla-sops/#generate-a-gpg-key)) |
| **Install and Configure Flux** <br>(Flux needs Git Repo Credentials & CSP IAM rights for KMS decryption or a kubernetes secret containing a private PGP decryption key)                                                                           | Presatisfied <br>(Demo Public Repo doesn't require RO Credentials, the demo PGP private decryption key is hosted cleartext in the repo)                                                                                                                                                                      | Prerequisite <br>(see BigBang docs, [flux docs](https://toolkit.fluxcd.io/components/source/gitrepositories/#spec-examples) are also a good resource for this)                                                             | Prerequisite <br>(see BigBang docs)                                                                                                                                                                 |
| **HTTPS Certificates**                                                                                                                                                                                                                            | Presatisfied <br>(Demo Public Repo contains a Let's Encrypt Free (public internet recognised certificate authority) HTTPS Certificate for *.bigbang.dev, alternatively mkcert can be used to generate demo certs for arbitrary DNS names that will only be trusted by the laptop that provisoned the mkcert) | Prerequisite <br>(HTTPS cert is provided by consumer)                                                                                                                                                                      | Prerequisite <br>(HTTPS cert is provided by consumer)                                                                                                                                               |
| **DNS**                                                                                                                                                                                                                                           | Edit your Laptop's host file (/etc/hosts, C:\Windows\System32\drivers\etc\hosts), or use something like AWS VPC Private DNS and [sshuttle](https://github.com/sshuttle/sshuttle) to point to host VM (if using k3d)                                                                                          | Prerequisite <br>(point DNS names to Layer 4 CSP LB)                                                                                                                                                                       | Prerequisite <br>(point DNS names to L4 LB)                                                                                                                                                         |
| **HTTPS Certificate, DNS Name, and hostnames in BigBang's helm values must match** <br>(in order for Ingress to work correctly.)                                                                                                                  | QuickStart leverages `*.bigbang.dev` HTTPS cert, and the BigBang Helm Chart's values.yaml's hostname defaults to bigbang.dev, just need to ensure multiple hostfile entries like "grafana.bigbang.dev " exist, or if you have access to DNS a wildcard entry to map CNAME `*.bigbang.dev` to k3d VM's IP     | Prerequisite <br>(update bigbang helm values in git repo so hostnames match HTTPS cert)                                                                                                                                    | Prerequisite <br>(update bigbang helm values in git repo so hostnames match HTTPS cert)                                                                                                             |
| **SSO Identity Provider** <br>(Prerequisite for SSO Authentication Proxy feature)                                                                                                                                                                 | Presatisfied* <br>(*depending on which quick start config is used), There's exists a demo SSO config that leverages P1's CAC enabled SSO, it's coded to only work for localhost to balance turn key demo functionality against security concerns.                                                            | Prerequisite <br>(You don't have to use Keycloak, you can use any OIDC/SAML Identity Provider) ([Customer Deployable Keycloak is a feature coming soon](https://repo1.dso.mil/platform-one/big-bang/bigbang/-/issues/291)) | Prerequisite* <br>(Install your own Keycloak cluster), leverage a pre-existing airgap SSO solution, or configure to not use SSO* if not needed for use case)                                        |
| **Ops Team to integrate, configure, and maintain BigBang** <br>(needed skillsets: DevOps IaC/CaC all the things, automate most the things, document the rest, linux administration, productionalization and maintenance of a Kubernetes Cluster.) | QuickStart Demo is designed to be self service.                                                                                                                                                                                                                                                              | Prerequisite <br>(BigBang Customer Integration Engineers are available to help long term Ops teams.)                                                                                                                       | Prerequisite                                                                                                                                                                                        |