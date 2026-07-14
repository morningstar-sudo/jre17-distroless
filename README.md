# java17-distroless

A base image containing **only a Java 17 runtime** (no application code): jlink trims the Temurin 17 JDK down to the required modules, placed on top of `cgr.dev/chainguard/glibc-dynamic` (Wolfi, distroless) — no shell, no package manager, runs as the `nonroot` user.

Trivy scan result: **0 CVEs** (all severities). The JRE only needs glibc (libstdc++ is statically linked into HotSpot, zlib is bundled by Temurin), so a glibc-only base is enough. Even the most minimal Debian base (`cc-debian12`) still carries ~17 unfixed glibc/gcc CVEs.

## Build & verify

```bash
docker build -t jre17-distroless .
docker run --rm jre17-distroless -version   # openjdk version "17.x"
```

## Customizing the JRE modules

The default module set covers most server applications (no `java.desktop`). To trim it exactly to your app:

```bash
jdeps --print-module-deps --ignore-missing-deps app.jar
docker build --build-arg JAVA_MODULES=<jdeps output> -t jre17-distroless .
```

## Using it as a base for your app

```dockerfile
FROM jre17-distroless
COPY app.jar /app/app.jar
CMD ["-jar", "/app/app.jar"]
```

## CVE scanning

```bash
trivy image --severity CRITICAL,HIGH,MEDIUM,LOW jre17-distroless
```

CI (`.github/workflows/build.yml`) builds the image and pushes it to the registry on `main`.

## Staying CVE-free over time

- Rebuild regularly: Wolfi and Temurin are patched continuously — new CVEs only disappear when you rebuild.
- For fully reproducible builds: pin both `FROM` images by digest (`@sha256:...`) and update them via Renovate/Dependabot.
- The Chainguard free tier only offers the `latest` tag — digest pinning matters even more if you need a fixed version.
- If you need `java.desktop` (AWT/Swing/imaging): it requires fontconfig/freetype — consider `gcr.io/distroless/java-base-debian12` (which carries the CVEs of its graphics libraries) or add the corresponding Wolfi packages.
