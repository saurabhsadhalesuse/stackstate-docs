---
description: SUSE Observability Self-hosted
---

# Self-signed certificates

## Overview

SUSE Observability has several points of interaction with external systems. For example, event handlers can call out to webhooks in other systems. With the default configuration, SUSE Observability won't be able to communicate with these systems if they're secured with TLS using a self-signed certificate, or a certificate that isn't by default trusted by the JVM.

To mitigate this, SUSE Observability allows configuration of a custom trust store.

## Create a custom trust store

You need to have the custom TLS certificate available. If you don't have that, you will need to [retrieve it via the browser](self-signed-certificates.md#retrieve-certificate-via-the-browser).

Use the keytool tool and the `cacerts` file included in the JVM \(Java Virtual Machine\) installation to convert an existing TLS certificate file to the format needed by SUSE Observability. You can run this on any machine, regardless of the type of operating system.

If you don't have the JVM installed on your computer, you can also [use a JVM Docker image](self-signed-certificates.md#using-a-docker-jvm) instead.

### Using an installed JVM

With the JVM installed on your computer and the certificate saved as a file `site.cert`, you can create a new trust store by taking the JVM's trust store and adding the extra certificate.

1. Create a working directory `workdir` and copy the certificate file `site.cert` to this directory.
2. Change directory to the `workdir` and make a copy of the `cacerts` file from your Java installation. `$JAVA_HOME` is an environment variable that contains the location of your Java installation. This is normally set when installing Java.

   ```bash
   cd workdir
   cp $JAVA_HOME/lib/security/cacerts ./custom_cacerts
   ```

3. Run the following keytool command to add the certificate. The required password is `changeit`. The alias needs to be a unique alias for the certificate, for example the domain name itself without any dots.

   ```bash
   keytool -import -keystore custom_cacerts -alias <a-name-for-the-certificate>  -file site.cert
   ```

4. The `custom_cacerts` store file will now include the `site.cert` certificate. You can verify that by searching for the alias in the output of

   ```bash
   keytool -list -keystore custom_cacerts
   ```

### Using a Docker JVM

If you don't have JVM installed on your computer, you can use a JVM Docker image. The certificate should be retrieved and saved as a file `site.cert`.

1. Create a working directory `workdir` and copy the certificate file `site.cert` to this directory.
2. Start the Java Docker container with the `workdir` mounted as a volume so it can be accessed:

   ```bash
   docker run -it -v `pwd`/workdir:/workdir  adoptopenjdk:11 bash
   ```

3. Change directory to the `workdir` and make a copy of the `cacerts` file:

   ```bash
   cd /workdir
   cp $JAVA_HOME/lib/security/cacerts ./custom_cacerts
   ```

4. Run the following keytool command to add the certificate. The required password is `changeit`. The alias needs to be a unique alias for the certificate, for example the domain name itself without any dots.

   ```bash
   keytool -import -keystore custom_cacerts -alias <a-name-for-the-certificate>  -file site.cert
   ```

5. The `custom_cacerts` store file will now include the `site.cert` certificate. You can verify that by searching for the alias in the output of

   ```bash
    keytool -list -keystore custom_cacerts
   ```

## Use a custom trust store

The trust store and the password can be specified as values. The trust store can only be specified from the helm command line as it's a file. The password value is specified in the same way in the example, but it can also be provided via a `values.yaml` file.

```bash
helm upgrade \
  --install \
  --namespace suse-observability \
  --values values.yaml \
  --set-file 'stackstate.java.trustStore'=custom_cacerts \
  --set 'stackstate.java.trustStorePassword'=changeit \
suse-observability \
suse-observability/suse-observability
```

{% hint style="info" %}
**Note:**

* The first run of the helm upgrade command will result in pods restarting, which may cause a short interruption of availability.
* Include these arguments on every `helm upgrade` run.
* The password and trust store are stored as a Kubernetes secret.
{% endhint %}

#### Base64 encoded trust stores

If needed, the Java trust store can also be configured by passing Base64 encoded strings into Helm values.

{% tabs %}
{% tab title="Linux" %}

To use a base64 encoded trust store, run the following `helm upgrade` command:

```bash
helm upgrade \
  --install \
  --namespace suse-observability \
  --values values.yaml \
  --set 'stackstate.java.trustStoreBase64Encoded'=$(cat custom_cacerts | base64 -w0) \
  --set 'stackstate.java.trustStorePassword'=changeit \
suse-observability \
suse-observability/suse-observability
```

{% endtab %}
{% tab title="MacOs" %}

To use a base64 encoded trust store, run the following `helm upgrade` command:

```bash
helm upgrade \
  --install \
  --namespace suse-observability \
  --values values.yaml \
  --set 'stackstate.java.trustStoreBase64Encoded'=$(cat custom_cacerts | base64) \
  --set 'stackstate.java.trustStorePassword'=changeit \
suse-observability \
suse-observability/suse-observability
```

{% endtab %}
{% endtabs %}

## Retrieve certificate via the browser

The certificate can be directly downloaded from the Chrome browser. The steps involved may vary slightly depending on the version you are using:

1. Navigate to the URL you need the certificate from.
2. Click the padlock icon in the location bar.
3. Click on **Certificate**.
4. Select **Details**.
5. Select **Export**.
6. Save using the default export file type \(Base64 ASCII encoded\).

