# Upgrade TAP `v1.3` to TAP `v1.4`

This user guide implements `Tanzu Application Platform` version upgrade from `v1.3` to `v1.4`.

- ref: https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.4/tap/upgrading.html

<!-- TOC -->
* [Upgrade TAP `v1.3` to TAP `v1.4`](#upgrade-tap-v13-to-tap-v14)
  * [Prerequisites:](#prerequisites-)
    * [1. `Kubernetes` cluster running TAP `v1.3`](#1-kubernetes-cluster-running-tap-v13)
    * [2. VMware registry credentials](#2-vmware-registry-credentials)
  * [Steps:](#steps-)
    * [Step 1 - Copy TAP `v1.4` image to your local image registry](#step-1---copy-tap-v14-image-to-your-local-image-registry)
        * [1. Login to `VMware` image registry](#1-login-to-vmware-image-registry)
        * [2. Import TAP `v1.4` image](#2-import-tap-v14-image)
    * [Step 2 - Update TAP package repository](#step-2---update-tap-package-repository)
        * [1. Update TAP's package repository and point it to the new TAP image.](#1-update-taps-package-repository-and-point-it-to-the-new-tap-image)
        * [2. Verify](#2-verify)
    * [Step 3 - Fix breaking changes via `tap-values.yaml` file](#step-3---fix-breaking-changes-via-tap-valuesyaml-file)
        * [1. Enable CVE scan results](#1-enable-cve-scan-results)
        * [2. TAP GUI serving `https`](#2-tap-gui-serving-https)
        * [3. Enable keyless support](#3-enable-keyless-support)
    * [Step 4 - Upgrade TAP package](#step-4---upgrade-tap-package)
    * [Step 5 - Verify TAP version](#step-5---verify-tap-version)
<!-- TOC -->

---

## Prerequisites:
##### 1. `Kubernetes` cluster running TAP `v1.3`
View TAP version
```bash
tanzu package installed get tap -n tap-install
```
**output:**
```bash
NAME:                    tap
PACKAGE-NAME:            tap.tanzu.vmware.com
PACKAGE-VERSION:         1.3.0
STATUS:                  Reconcile succeeded
CONDITIONS:              [{ReconcileSucceeded True  }]
USEFUL-ERROR-MESSAGE:    
```

##### 2. VMware registry credentials
Accessing `VMware`'s image repository requires login to pull TAP v1.4 image bundle.
Please make sure to have your `<tanzunet_username>` and `<tanzunet_password>` available.
---
## Steps:
### Step 1 - Copy TAP `v1.4` image to your local image registry
##### 1. Login to `VMware` image registry
Insert your username and password in the command prompt
```bash
docker login registry.tanzu.vmware.com
```
**output:**
```bash
Login Succeeded
```

##### 2. Import TAP `v1.4` image
```bash
imgpkg copy --concurrency 1 -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.4.0 --to-repo <image-registry-domain-name>/<repo-name>

#for example
imgpkg copy --concurrency 1 -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.4.0 --to-repo 384617649113.dkr.ecr.eu-north-1.amazonaws.com/p8m8nm3rts18/tanzu-application-platform/tap-packages
```
**output:**
```bash
[...]
copy | exported 214 images
copy | importing 214 images...

6.96 GiB / 6.96 GiB [----------------------------------------------------------------------------------------------------->] 100.00% 103.68 MiB p/s
copy | 
copy | done uploading images
copy | Warning: Skipped the followings layer(s) due to it being non-distributable. If you would like to include non-distributable layers, use the --include-non-distributable-layers flag
copy |  - Image: 384617649113.dkr.ecr.eu-north-1.amazonaws.com/p8m8nm3rts18/tanzu-application-platform/tap-packages@sha256:0d8a465e660d9cd4ea4f070813bc9b8ef83b9fa8345617ea12810db0c068e8bf
copy |    Layers:
copy |      - sha256:5c9d6483dab113d2d9b50fdf3156622aa2ec3d6faaed5664d15a5568032d1714
copy |  - Image: 384617649113.dkr.ecr.eu-north-1.amazonaws.com/p8m8nm3rts18/tanzu-application-platform/tap-packages@sha256:c4961a33a4c6e269705792e58d175f06410268410bcffff85461144dd823b61c
copy |    Layers:
copy |      - sha256:5c9d6483dab113d2d9b50fdf3156622aa2ec3d6faaed5664d15a5568032d1714
copy | Tagging images
```

### Step 2 - Update TAP package repository 
##### 1. Update TAP's package repository and point it to the new TAP image.
```bash
tanzu package repository update tanzu-tap-repository \
    --url <image-registry-domain-name>/<repo-name>:1.4.0  \
    --namespace tap-install
    
#for example
tanzu package repository update tanzu-tap-repository \
    --url 384617649113.dkr.ecr.eu-north-1.amazonaws.com/p8m8nm3rts18/tanzu-application-platform/tap-packages:1.4.0  \
    --namespace tap-install
```
**output:**
```bash
Updating package repository 'tanzu-tap-repository'
Getting package repository 'tanzu-tap-repository'
Validating provided settings for the package repository
Updating package repository resource
Waiting for 'PackageRepository' reconciliation for 'tanzu-tap-repository'
'PackageRepository' resource install status: Reconciling
'PackageRepository' resource install status: ReconcileSucceeded
Updated package repository 'tanzu-tap-repository' in namespace 'tap-install'
```

##### 2. Verify
```bash
tanzu package repository get tanzu-tap-repository --namespace tap-install
```
**output:**
```bash
NAME:          tanzu-tap-repository
VERSION:       150434914
REPOSITORY:    384617649113.dkr.ecr.eu-north-1.amazonaws.com/p8m8nm3rts18/tanzu-application-platform/tap-packages
TAG:           1.4.0
STATUS:        Reconcile succeeded
REASON:        
```

### Step 3 - Fix breaking changes via `tap-values.yaml` file
- ref: https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.4/tap/release-notes.html#breaking-changes-3

##### 1. Enable CVE scan results
- ref: https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.4/tap/tap-gui-plugins-scc-tap-gui.html#scan

In previous versions of `Tanzu Application Platform`, you configured TAP GUI to use the `read-only` access token to communicate with Supply Chain Security Tools - Store.

In `v1.4`, you must use the `read-write` access token to use new features in the Security Analysis GUI plug-in. 

1. Retrieve the `read-write` token, which is created by default when installing `Tanzu Application Platform`.
```bash
kubectl get secrets metadata-store-read-write-client -n metadata-store -o jsonpath="{.data.token}" | base64 -d
```
2. Add this proxy configuration or replace the current Bearer token of the `tap-gui` section of `tap-values.yaml`:
```yaml
tap_gui:
  app_config:
    proxy:
      /metadata-store:
        target: https://metadata-store-app.metadata-store:8443/api/v1
        changeOrigin: true
        secure: false
        headers:
          Authorization: "Bearer READ-WRITE-ACCESS-TOKEN"
          X-Custom-Source: project-star 
```

##### 2. TAP GUI serving `https`
- ref: https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.4/tap/tap-gui-troubleshooting.html#catalog-not-loading

As of `v1.4`, TAP GUI provides TLS connections by default. Because of this, if you visit a `Tanzu Application Platform` GUI site, your connection is automatically upgraded to `https`.

You might have manually set the fields `app.baseUrl`, `backend.baseUrl`, and `backend.cors.origin` in your `tap-values.yaml` file. 

TAP GUI uses the `baseUrl` to determine how to create links to fetch from its APIs. 
The combination of these two factors causes your browser to attempt to fetch mixed-content.

The solution is to delete these fields or update your values in `tap-values.yaml` to reflect that your TAP GUI instance is serving `https`, as in the following example:
```yaml
tap_gui:
  app_config:
    app:
      baseUrl: https://tap-gui.INGRESS-DOMAIN/
    backend:
      baseUrl: https://tap-gui.INGRESS-DOMAIN/
      cors:
        origin: https://tap-gui.INGRESS-DOMAIN/
```


##### 3. Enable keyless support
In TAP `v1.4`, keyless support is deactivated by default.

To support the keyless authorities in `ClusterImagePolicy`, Policy Controller no longer initializes TUF by default. 

To continue using keyless authorities, you must set the `policy.tuf_enabled` field to `true` in the `tap-values.yaml` file during the upgrade process.
```yaml
policy:
  tuf_enabled: true
```

### Step 4 - Upgrade TAP package

```bash
tanzu package installed update tap --version 1.4.0 --values-file ~/<PATH-TO-FILE>/tap-values.yaml -n tap-install 

#for example
tanzu package installed update tap --version 1.4.0 --values-file ~/tap-setup-scripts/generated/tap-values.yaml -n tap-install 
```
**output:**
```bash
Updating installed package 'tap'
Getting package install for 'tap'
Getting package metadata for 'tap.tanzu.vmware.com'
Updating secret 'tap-tap-install-values'
Updating package install for 'tap'
Waiting for 'PackageInstall' reconciliation for 'tap'
'PackageInstall' resource install status: ReconcileSucceeded
Updated installed package 'tap' in namespace 'tap-install'
```

### Step 5 - Verify TAP version
```bash
tanzu package installed list -A
```
**output:**
```bash
NAME                                PACKAGE-NAME                                         PACKAGE-VERSION  STATUS               NAMESPACE    
  accelerator                         accelerator.apps.tanzu.vmware.com                    1.4.0            Reconcile succeeded  tap-install  
  api-auto-registration               apis.apps.tanzu.vmware.com                           0.2.1            Reconcile succeeded  tap-install  
  api-portal                          api-portal.tanzu.vmware.com                          1.2.6            Reconcile succeeded  tap-install  
  appliveview                         backend.appliveview.tanzu.vmware.com                 1.4.0            Reconcile succeeded  tap-install  
  appliveview-connector               connector.appliveview.tanzu.vmware.com               1.4.0            Reconcile succeeded  tap-install  
  appliveview-conventions             conventions.appliveview.tanzu.vmware.com             1.4.0            Reconcile succeeded  tap-install  
  appsso                              sso.apps.tanzu.vmware.com                            3.0.0            Reconcile succeeded  tap-install  
  buildservice                        buildservice.tanzu.vmware.com                        1.9.0            Reconcile succeeded  tap-install  
  cartographer                        cartographer.tanzu.vmware.com                        0.6.2            Reconcile succeeded  tap-install  
  cert-manager                        cert-manager.tanzu.vmware.com                        2.0.0            Reconcile succeeded  tap-install  
  cnrs                                cnrs.tanzu.vmware.com                                2.1.0            Reconcile succeeded  tap-install  
  contour                             contour.tanzu.vmware.com                             1.22.3+tap.2     Reconcile succeeded  tap-install  
  conventions-controller              controller.conventions.apps.tanzu.vmware.com         0.8.0            Reconcile succeeded  tap-install  
  developer-conventions               developer-conventions.tanzu.vmware.com               0.9.0            Reconcile succeeded  tap-install  
  eventing                            eventing.tanzu.vmware.com                            2.1.1            Reconcile succeeded  tap-install  
  fluxcd-source-controller            fluxcd.source.controller.tanzu.vmware.com            0.27.0+tap.4     Reconcile succeeded  tap-install  
  grype                               grype.scanning.apps.tanzu.vmware.com                 1.4.0            Reconcile succeeded  tap-install  
  learningcenter                      learningcenter.tanzu.vmware.com                      0.2.5            Reconcile succeeded  tap-install  
  learningcenter-workshops            workshops.learningcenter.tanzu.vmware.com            0.2.4            Reconcile succeeded  tap-install  
  metadata-store                      metadata-store.apps.tanzu.vmware.com                 1.4.2            Reconcile succeeded  tap-install  
  namespace-provisioner               namespace-provisioner.apps.tanzu.vmware.com          0.1.2            Reconcile succeeded  tap-install  
  ootb-delivery-basic                 ootb-delivery-basic.tanzu.vmware.com                 0.11.0           Reconcile succeeded  tap-install  
  ootb-supply-chain-testing-scanning  ootb-supply-chain-testing-scanning.tanzu.vmware.com  0.11.0           Reconcile succeeded  tap-install  
  ootb-templates                      ootb-templates.tanzu.vmware.com                      0.11.0           Reconcile succeeded  tap-install  
  scanning                            scanning.apps.tanzu.vmware.com                       1.4.0            Reconcile succeeded  tap-install  
  service-bindings                    service-bindings.labs.vmware.com                     0.9.0            Reconcile succeeded  tap-install  
  services-toolkit                    services-toolkit.tanzu.vmware.com                    0.9.0            Reconcile succeeded  tap-install  
  source-controller                   controller.source.apps.tanzu.vmware.com              0.6.0            Reconcile succeeded  tap-install  
  spring-boot-conventions             spring-boot-conventions.tanzu.vmware.com             1.4.0            Reconcile succeeded  tap-install  
  tap                                 tap.tanzu.vmware.com                                 1.4.0            Reconcile succeeded  tap-install  
  tap-auth                            tap-auth.tanzu.vmware.com                            1.1.0            Reconcile succeeded  tap-install  
  tap-gui                             tap-gui.tanzu.vmware.com                             1.4.3            Reconcile succeeded  tap-install  
  tap-telemetry                       tap-telemetry.tanzu.vmware.com                       0.4.1-build.2    Reconcile succeeded  tap-install  
  tekton-pipelines                    tekton.tanzu.vmware.com                              0.41.0+tap.4     Reconcile succeeded  tap-install
```
View tap package:
```bash
tanzu package installed get tap -n tap-install
```
**output:**
```bash
NAME:                    tap
PACKAGE-NAME:            tap.tanzu.vmware.com
PACKAGE-VERSION:         1.4.0
STATUS:                  Reconcile succeeded
CONDITIONS:              [{ReconcileSucceeded True  }]
```
