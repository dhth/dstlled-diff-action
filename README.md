<p align="center">
  <h1 align="center">dstlled-diff</h1>
  <p align="center">
    <a href="https://github.com/dhth/dstlled-diff-action/releases/latest"><img alt="GitHub release" src="https://img.shields.io/github/release/dhth/dstlled-diff-action.svg?logo=github&style=flat-square"></a>
    <a href="https://github.com/marketplace/actions/dstlled-diff"><img alt="GitHub marketplace" src="https://img.shields.io/badge/marketplace-dstlled--diff--action-blue?logo=github&style=flat-square"></a>
    <a href="https://dhth.github.io/dstlled-diff-action"><img alt="GitHub marketplace" src="https://img.shields.io/website?url=https%3A%2F%2Fdhth.github.io%2Fdstlled-diff-action&style=flat-square&label=web-demo"></a>
  </p>
</p>

✨ Overview
---

`dstlled-diff` (short for "distilled-diff") is a GitHub action that processes a
specific git revision range and generates a diff that only includes changes in
the signatures of "code constructs" *(functions, methods, classes, traits,
interfaces, objects, type aliases, enums, etc.)*. It is powered by [dstll][1].

The main goal of `dstlled-diff` is to simplify the review of large structural
code changes by removing diff components that do not alter signatures.

![Usage](https://tools.dhruvs.space/images/dstlled-diff/dstlled-diff-1.png)

📜 Languages supported
---

- ![go](https://img.shields.io/badge/go-grey?logo=go)
- ![python](https://img.shields.io/badge/python-grey?logo=python)
- ![rust](https://img.shields.io/badge/rust-grey?logo=rust)
- ![scala 2](https://img.shields.io/badge/scala-grey?logo=scala)
- more to come

Δ Difference to regular git diff
---

👉 Consider this *(fairly long)* git diff:

<details><summary> expand </summary>

```diff
diff --git a/.github/workflows/build.yml b/.github/workflows/build.yml
index bb98703..b1b2a5c 100644
--- a/.github/workflows/build.yml
+++ b/.github/workflows/build.yml
@@ -1,35 +1,31 @@
 name: build
 
 on:
   push:
-    branches: [ "main" ]
+    branches: ["main"]
   pull_request:
     paths:
       - "go.*"
       - "**/*.go"
       - ".github/workflows/*.yml"
 
 permissions:
   contents: read
 
 env:
-  GO_VERSION: '1.23.0'
+  GO_VERSION: '1.23.1'
 
 jobs:
   build:
     strategy:
       matrix:
         os: [ubuntu-latest, macos-latest]
     runs-on: ${{ matrix.os }}
     steps:
-    - uses: actions/checkout@v4
-    - name: Set up Go
-      uses: actions/setup-go@v5
-      with:
-        go-version: ${{ env.GO_VERSION }}
-    - name: go build
-      run: go build -v ./...
-    - name: golangci-lint
-      uses: golangci/golangci-lint-action@v6
-      with:
-        version: v1.60
+      - uses: actions/checkout@v4
+      - name: Set up Go
+        uses: actions/setup-go@v5
+        with:
+          go-version: ${{ env.GO_VERSION }}
+      - name: go build
+        run: go build -v ./...
diff --git a/.github/workflows/dstlled-diff.yml b/.github/workflows/dstlled-diff.yml
new file mode 100644
index 0000000..7cbfcca
--- /dev/null
+++ b/.github/workflows/dstlled-diff.yml
@@ -0,0 +1,24 @@
+name: dstlled-diff
+
+on:
+  pull_request:
+    types: [opened, reopened, synchronize, ready_for_review]
+
+permissions:
+  contents: read
+  pull-requests: write
+
+jobs:
+  dstlled-diff:
+    runs-on: ubuntu-latest
+    steps:
+      - uses: actions/checkout@v4
+        with:
+          fetch-depth: 0
+      - id: get-dstlled-diff
+        uses: dhth/dstlled-diff-action@v0.1.2
+        with:
+          pattern: '**.go'
+          starting-commit: ${{ github.event.pull_request.base.sha }}
+          ending-commit: ${{ github.event.pull_request.head.sha }}
+          post-comment-on-pr: 'true'
diff --git a/.github/workflows/lint.yml b/.github/workflows/lint.yml
new file mode 100644
index 0000000..bbd8e5d
--- /dev/null
+++ b/.github/workflows/lint.yml
@@ -0,0 +1,30 @@
+name: lint
+
+on:
+  push:
+    branches: ["main"]
+  pull_request:
+    paths:
+      - "go.*"
+      - "**/*.go"
+      - ".github/workflows/*.yml"
+
+permissions:
+  contents: read
+
+env:
+  GO_VERSION: '1.23.1'
+
+jobs:
+  lint:
+    runs-on: ubuntu-latest
+    steps:
+      - uses: actions/checkout@v4
+      - name: Set up Go
+        uses: actions/setup-go@v5
+        with:
+          go-version: ${{ env.GO_VERSION }}
+      - name: golangci-lint
+        uses: golangci/golangci-lint-action@v6
+        with:
+          version: v1.60
diff --git a/.github/workflows/release.yml b/.github/workflows/release.yml
index a9fe1c0..a2f01bb 100644
--- a/.github/workflows/release.yml
+++ b/.github/workflows/release.yml
@@ -1,40 +1,39 @@
 name: release
 
 on:
   push:
     tags:
       - 'v*'
 
+permissions:
+  id-token: write
+
 env:
-  GO_VERSION: '1.23.0'
+  GO_VERSION: '1.23.1'
 
 jobs:
-  build:
+  release:
     runs-on: ubuntu-latest
     steps:
-    - uses: actions/checkout@v4
-      with:
-        fetch-depth: 0
-    - name: Set up Go
-      uses: actions/setup-go@v5
-      with:
-        go-version: ${{ env.GO_VERSION }}
-    - name: Build
-      run: go build -v ./...
-    - name: Install Cosign
-      uses: sigstore/cosign-installer@v3
-      with:
-        cosign-release: 'v2.2.3'
-    - name: Store Cosign private key in a file
-      run: 'echo "$COSIGN_KEY" > cosign.key'
-      shell: bash
-      env:
-        COSIGN_KEY: ${{secrets.COSIGN_KEY}}
-    - name: Release Binaries
-      uses: goreleaser/goreleaser-action@v6
-      with:
-        version: latest
-        args: release --clean
-      env:
-        GITHUB_TOKEN: ${{secrets.GH_PAT}}
-        COSIGN_PASSWORD: ${{secrets.COSIGN_PASSWORD}}
+      - uses: actions/checkout@v4
+        with:
+          fetch-depth: 0
+      - name: Set up Go
+        uses: actions/setup-go@v5
+        with:
+          go-version: ${{ env.GO_VERSION }}
+      - name: Build
+        run: go build -v ./...
+      - name: Test
+        run: go test -v ./...
+      - name: Install Cosign
+        uses: sigstore/cosign-installer@v3
+        with:
+          cosign-release: 'v2.2.3'
+      - name: Release Binaries
+        uses: goreleaser/goreleaser-action@v6
+        with:
+          version: latest
+          args: release --clean
+        env:
+          GITHUB_TOKEN: ${{secrets.GH_PAT}}
diff --git a/.github/workflows/vulncheck.yml b/.github/workflows/vulncheck.yml
index 092ed53..db473db 100644
--- a/.github/workflows/vulncheck.yml
+++ b/.github/workflows/vulncheck.yml
@@ -1,32 +1,31 @@
 name: vulncheck
 on:
   push:
-    branches: [ "main" ]
+    branches: ["main"]
   pull_request:
     paths:
       - "go.*"
       - "**/*.go"
       - ".github/workflows/*.yml"
 
 permissions:
   contents: read
 
 env:
-  GO_VERSION: '1.23.0'
+  GO_VERSION: '1.23.1'
 
 jobs:
   vulncheck:
     name: vulncheck
     runs-on: ubuntu-latest
     steps:
-    - uses: actions/checkout@v4
-    - name: Set up Go
-      uses: actions/setup-go@v5
-      with:
-        go-version: ${{ env.GO_VERSION }}
-    - name: govulncheck
-      shell: bash
-      run: |
-        go install golang.org/x/vuln/cmd/govulncheck@latest
-        govulncheck ./...
-
+      - uses: actions/checkout@v4
+      - name: Set up Go
+        uses: actions/setup-go@v5
+        with:
+          go-version: ${{ env.GO_VERSION }}
+      - name: govulncheck
+        shell: bash
+        run: |
+          go install golang.org/x/vuln/cmd/govulncheck@latest
+          govulncheck ./...
diff --git a/.golangci.yml b/.golangci.yml
new file mode 100644
index 0000000..12df18a
--- /dev/null
+++ b/.golangci.yml
@@ -0,0 +1,60 @@
+linters:
+  enable:
+    - errcheck
+    - errname
+    - errorlint
+    - goconst
+    - gofumpt
+    - gosimple
+    - govet
+    - ineffassign
+    - nilerr
+    - prealloc
+    - predeclared
+    - revive
+    - rowserrcheck
+    - sqlclosecheck
+    - staticcheck
+    - testifylint
+    - thelper
+    - unconvert
+    - unused
+    - usestdlibvars
+    - wastedassign
+linters-settings:
+  revive:
+    rules:
+      # defaults
+      - name: blank-imports
+      - name: context-as-argument
+        arguments:
+          - allowTypesBefore: "*testing.T"
+      - name: context-keys-type
+      - name: dot-imports
+      - name: empty-block
+      - name: error-naming
+      - name: error-return
+      - name: error-strings
+      - name: errorf
+      - name: exported
+      - name: if-return
+      - name: increment-decrement
+      - name: indent-error-flow
+      - name: package-comments
+      - name: range
+      - name: receiver-naming
+      - name: redefines-builtin-id
+      - name: superfluous-else
+      - name: time-naming
+      - name: unexported-return
+      - name: unreachable-code
+      - name: unused-parameter
+      - name: var-declaration
+      - name: var-naming
+      # additional
+      - name: unnecessary-stmt
+      - name: deep-exit
+      - name: confusing-naming
+      - name: unused-receiver
+      - name: unhandled-error
+        arguments: ["fmt.Print", "fmt.Println", "fmt.Printf", "fmt.Fprintf", "fmt.Fprint"]
diff --git a/.goreleaser.yaml b/.goreleaser.yml
similarity index 74%
rename from .goreleaser.yaml
rename to .goreleaser.yml
index 689a601..925536d 100644
--- a/.goreleaser.yaml
+++ b/.goreleaser.yml
@@ -1,46 +1,46 @@
 version: 2
 
 before:
   hooks:
     - go mod tidy
-    - go generate ./...
 
 builds:
   - env:
       - CGO_ENABLED=0
     goos:
       - linux
       - darwin
     goarch:
       - amd64
       - arm64
+
 signs:
   - cmd: cosign
-    stdin: "{{ .Env.COSIGN_PASSWORD }}"
+    signature: "${artifact}.sig"
+    certificate: "${artifact}.pem"
     args:
       - "sign-blob"
-      - "--key=cosign.key"
+      - "--oidc-issuer=https://token.actions.githubusercontent.com"
+      - "--output-certificate=${certificate}"
       - "--output-signature=${signature}"
       - "${artifact}"
-      - "--yes" # needed on cosign 2.0.0+
-    artifacts: all
-
+      - "--yes"
+    artifacts: checksum
 
 brews:
   - name: cueitup
     repository:
       owner: dhth
       name: homebrew-tap
     directory: Formula
     license: MIT
     homepage: "https://github.com/dhth/cueitup"
     description: "Inspect messages in an AWS SQS queue in a simple and deliberate manner"
 
 changelog:
   sort: asc
   filters:
     exclude:
       - "^docs:"
       - "^test:"
       - "^ci:"
-
diff --git a/cmd/root.go b/cmd/root.go
index 9bdc5c8..8d40f9b 100644
--- a/cmd/root.go
+++ b/cmd/root.go
@@ -1,89 +1,94 @@
 package cmd
 
 import (
 	"context"
+	"errors"
 	"flag"
 	"fmt"
 	"os"
 	"strings"
 
 	"github.com/aws/aws-sdk-go-v2/config"
 	"github.com/aws/aws-sdk-go-v2/service/sqs"
 	"github.com/dhth/cueitup/ui"
 	"github.com/dhth/cueitup/ui/model"
 )
 
-func die(msg string, args ...any) {
-	fmt.Fprintf(os.Stderr, msg+"\n", args...)
-	os.Exit(1)
-}
+var (
+	errQueueURLEmpty        = errors.New("queue URL is empty")
+	errAWSProfileEmpty      = errors.New("AWS profile is empty")
+	errAWSRegionEmpty       = errors.New("AWS region is empty")
+	errQueueURLIncorrect    = errors.New("queue URL is incorrect")
+	errMsgFormatInvalid     = errors.New("message format is invalid")
+	errInvalidFlagUsage     = errors.New("invalid flag usage")
+	errCouldntLoadAWSConfig = errors.New("couldn't load AWS config")
+)
 
 var (
-	queueUrl   = flag.String("queue-url", "", "url of the queue to consume from")
+	queueURL   = flag.String("queue-url", "", "url of the queue to consume from")
 	awsProfile = flag.String("aws-profile", "", "aws profile to use")
 	awsRegion  = flag.String("aws-region", "", "aws region to use")
 	msgFormat  = flag.String("msg-format", "json", "message format")
 	subsetKey  = flag.String("subset-key", "", "extract a nested object inside the JSON body")
 	contextKey = flag.String("context-key", "", "the key to use as for context in the list")
 )
 
-func Execute() {
-
+func Execute() error {
 	flag.Usage = func() {
 		fmt.Fprintf(os.Stderr, "Inspect messages in an AWS SQS queue in a simple and deliberate manner.\n\nFlags:\n")
 		flag.PrintDefaults()
 		fmt.Fprintf(os.Stderr, "\n------\n%s", model.HelpText)
 	}
 	flag.Parse()
 
-	if *queueUrl == "" {
-		die("queue-url cannot be empty")
-	} else if !strings.HasPrefix(*queueUrl, "https://") {
-		die("queue-url must begin with https")
+	if *queueURL == "" {
+		return errQueueURLEmpty
+	}
+
+	if !strings.HasPrefix(*queueURL, "https://") {
+		return fmt.Errorf("%w: must begin with https", errQueueURLIncorrect)
 	}
 
 	if *awsProfile == "" {
-		die("aws-profile cannot be empty")
+		return errAWSProfileEmpty
 	}
 
 	if *awsRegion == "" {
-		die("aws-region cannot be empty")
+		return errAWSRegionEmpty
 	}
 
 	var msgFmt model.MsgFmt
 	switch *msgFormat {
 	case "json":
-		msgFmt = model.JsonFmt
+		msgFmt = model.JSONFmt
 	case "plaintext":
 		msgFmt = model.PlainTxtFmt
 	default:
-		die("cueitup only supports the following msg-format values: json, plaintext")
+		return fmt.Errorf("%w: supported values: json, plaintext", errMsgFormatInvalid)
 	}
 
-	if *subsetKey != "" && msgFmt != model.JsonFmt {
-		die("subset-key can only be used when msg-format=json")
+	if *subsetKey != "" && msgFmt != model.JSONFmt {
+		return fmt.Errorf("%w: subset-key can only be used when msg-format=json", errInvalidFlagUsage)
 	}
-	if *contextKey != "" && msgFmt != model.JsonFmt {
-		die("context-key can only be used when msg-format=json")
+	if *contextKey != "" && msgFmt != model.JSONFmt {
+		return fmt.Errorf("%w: context-key can only be used when msg-format=json", errInvalidFlagUsage)
 	}
 
 	msgConsumptionConf := model.MsgConsumptionConf{
 		Format:     msgFmt,
 		SubsetKey:  *subsetKey,
 		ContextKey: *contextKey,
 	}
 
 	sdkConfig, err := config.LoadDefaultConfig(context.TODO(),
 		config.WithSharedConfigProfile(*awsProfile),
 		config.WithRegion(*awsRegion),
 	)
 	if err != nil {
-		fmt.Println("Error:", err)
-		os.Exit(1)
+		return fmt.Errorf("%w: %s", errCouldntLoadAWSConfig, err.Error())
 	}
 
 	sqsClient := sqs.NewFromConfig(sdkConfig)
 
-	ui.RenderUI(sqsClient, *queueUrl, msgConsumptionConf)
-
+	return ui.RenderUI(sqsClient, *queueURL, msgConsumptionConf)
 }
diff --git a/go.mod b/go.mod
index 32d834d..4ab9ba2 100644
--- a/go.mod
+++ b/go.mod
@@ -1,44 +1,44 @@
 module github.com/dhth/cueitup
 
-go 1.23.0
+go 1.23.1
 
 require (
 	github.com/aws/aws-sdk-go-v2 v1.30.5
-	github.com/aws/aws-sdk-go-v2/config v1.27.32
-	github.com/aws/aws-sdk-go-v2/service/sqs v1.34.7
-	github.com/charmbracelet/bubbles v0.19.0
-	github.com/charmbracelet/bubbletea v1.1.0
+	github.com/aws/aws-sdk-go-v2/config v1.27.33
+	github.com/aws/aws-sdk-go-v2/service/sqs v1.34.8
+	github.com/charmbracelet/bubbles v0.20.0
+	github.com/charmbracelet/bubbletea v1.1.1
 	github.com/charmbracelet/lipgloss v0.13.0
 	github.com/tidwall/pretty v1.2.1
 )
 
 require (
 	github.com/atotto/clipboard v0.1.4 // indirect
-	github.com/aws/aws-sdk-go-v2/credentials v1.17.31 // indirect
+	github.com/aws/aws-sdk-go-v2/credentials v1.17.32 // indirect
 	github.com/aws/aws-sdk-go-v2/feature/ec2/imds v1.16.13 // indirect
 	github.com/aws/aws-sdk-go-v2/internal/configsources v1.3.17 // indirect
 	github.com/aws/aws-sdk-go-v2/internal/endpoints/v2 v2.6.17 // indirect
 	github.com/aws/aws-sdk-go-v2/internal/ini v1.8.1 // indirect
 	github.com/aws/aws-sdk-go-v2/service/internal/accept-encoding v1.11.4 // indirect
 	github.com/aws/aws-sdk-go-v2/service/internal/presigned-url v1.11.19 // indirect
-	github.com/aws/aws-sdk-go-v2/service/sso v1.22.6 // indirect
-	github.com/aws/aws-sdk-go-v2/service/ssooidc v1.26.6 // indirect
-	github.com/aws/aws-sdk-go-v2/service/sts v1.30.6 // indirect
+	github.com/aws/aws-sdk-go-v2/service/sso v1.22.7 // indirect
+	github.com/aws/aws-sdk-go-v2/service/ssooidc v1.26.7 // indirect
+	github.com/aws/aws-sdk-go-v2/service/sts v1.30.7 // indirect
 	github.com/aws/smithy-go v1.20.4 // indirect
 	github.com/aymanbagabas/go-osc52/v2 v2.0.1 // indirect
-	github.com/charmbracelet/x/ansi v0.2.3 // indirect
+	github.com/charmbracelet/x/ansi v0.3.1 // indirect
 	github.com/charmbracelet/x/term v0.2.0 // indirect
 	github.com/erikgeiser/coninput v0.0.0-20211004153227-1c3628e74d0f // indirect
 	github.com/lucasb-eyer/go-colorful v1.2.0 // indirect
 	github.com/mattn/go-isatty v0.0.20 // indirect
 	github.com/mattn/go-localereader v0.0.1 // indirect
 	github.com/mattn/go-runewidth v0.0.16 // indirect
 	github.com/muesli/ansi v0.0.0-20230316100256-276c6243b2f6 // indirect
 	github.com/muesli/cancelreader v0.2.2 // indirect
 	github.com/muesli/termenv v0.15.2 // indirect
 	github.com/rivo/uniseg v0.4.7 // indirect
 	github.com/sahilm/fuzzy v0.1.1 // indirect
 	golang.org/x/sync v0.8.0 // indirect
-	golang.org/x/sys v0.24.0 // indirect
-	golang.org/x/text v0.17.0 // indirect
+	golang.org/x/sys v0.25.0 // indirect
+	golang.org/x/text v0.18.0 // indirect
 )
diff --git a/go.sum b/go.sum
index d44d80e..7c908a1 100644
--- a/go.sum
+++ b/go.sum
@@ -1,50 +1,50 @@
 github.com/atotto/clipboard v0.1.4 h1:EH0zSVneZPSuFR11BlR9YppQTVDbh5+16AmcJi4g1z4=
 github.com/atotto/clipboard v0.1.4/go.mod h1:ZY9tmq7sm5xIbd9bOK4onWV4S6X0u6GY7Vn0Yu86PYI=
 github.com/aws/aws-sdk-go-v2 v1.30.5 h1:mWSRTwQAb0aLE17dSzztCVJWI9+cRMgqebndjwDyK0g=
 github.com/aws/aws-sdk-go-v2 v1.30.5/go.mod h1:CT+ZPWXbYrci8chcARI3OmI/qgd+f6WtuLOoaIA8PR0=
-github.com/aws/aws-sdk-go-v2/config v1.27.32 h1:jnAMVTJTpAQlePCUUlnXnllHEMGVWmvUJOiGjgtS9S0=
-github.com/aws/aws-sdk-go-v2/config v1.27.32/go.mod h1:JibtzKJoXT0M/MhoYL6qfCk7nm/MppwukDFZtdgVRoY=
-github.com/aws/aws-sdk-go-v2/credentials v1.17.31 h1:jtyfcOfgoqWA2hW/E8sFbwdfgwD3APnF9CLCKE8dTyw=
-github.com/aws/aws-sdk-go-v2/credentials v1.17.31/go.mod h1:RSgY5lfCfw+FoyKWtOpLolPlfQVdDBQWTUniAaE+NKY=
+github.com/aws/aws-sdk-go-v2/config v1.27.33 h1:Nof9o/MsmH4oa0s2q9a0k7tMz5x/Yj5k06lDODWz3BU=
+github.com/aws/aws-sdk-go-v2/config v1.27.33/go.mod h1:kEqdYzRb8dd8Sy2pOdEbExTTF5v7ozEXX0McgPE7xks=
+github.com/aws/aws-sdk-go-v2/credentials v1.17.32 h1:7Cxhp/BnT2RcGy4VisJ9miUPecY+lyE9I8JvcZofn9I=
+github.com/aws/aws-sdk-go-v2/credentials v1.17.32/go.mod h1:P5/QMF3/DCHbXGEGkdbilXHsyTBX5D3HSwcrSc9p20I=
 github.com/aws/aws-sdk-go-v2/feature/ec2/imds v1.16.13 h1:pfQ2sqNpMVK6xz2RbqLEL0GH87JOwSxPV2rzm8Zsb74=
 github.com/aws/aws-sdk-go-v2/feature/ec2/imds v1.16.13/go.mod h1:NG7RXPUlqfsCLLFfi0+IpKN4sCB9D9fw/qTaSB+xRoU=
 github.com/aws/aws-sdk-go-v2/internal/configsources v1.3.17 h1:pI7Bzt0BJtYA0N/JEC6B8fJ4RBrEMi1LBrkMdFYNSnQ=
 github.com/aws/aws-sdk-go-v2/internal/configsources v1.3.17/go.mod h1:Dh5zzJYMtxfIjYW+/evjQ8uj2OyR/ve2KROHGHlSFqE=
 github.com/aws/aws-sdk-go-v2/internal/endpoints/v2 v2.6.17 h1:Mqr/V5gvrhA2gvgnF42Zh5iMiQNcOYthFYwCyrnuWlc=
 github.com/aws/aws-sdk-go-v2/internal/endpoints/v2 v2.6.17/go.mod h1:aLJpZlCmjE+V+KtN1q1uyZkfnUWpQGpbsn89XPKyzfU=
 github.com/aws/aws-sdk-go-v2/internal/ini v1.8.1 h1:VaRN3TlFdd6KxX1x3ILT5ynH6HvKgqdiXoTxAF4HQcQ=
 github.com/aws/aws-sdk-go-v2/internal/ini v1.8.1/go.mod h1:FbtygfRFze9usAadmnGJNc8KsP346kEe+y2/oyhGAGc=
 github.com/aws/aws-sdk-go-v2/service/internal/accept-encoding v1.11.4 h1:KypMCbLPPHEmf9DgMGw51jMj77VfGPAN2Kv4cfhlfgI=
 github.com/aws/aws-sdk-go-v2/service/internal/accept-encoding v1.11.4/go.mod h1:Vz1JQXliGcQktFTN/LN6uGppAIRoLBR2bMvIMP0gOjc=
 github.com/aws/aws-sdk-go-v2/service/internal/presigned-url v1.11.19 h1:rfprUlsdzgl7ZL2KlXiUAoJnI/VxfHCvDFr2QDFj6u4=
 github.com/aws/aws-sdk-go-v2/service/internal/presigned-url v1.11.19/go.mod h1:SCWkEdRq8/7EK60NcvvQ6NXKuTcchAD4ROAsC37VEZE=
-github.com/aws/aws-sdk-go-v2/service/sqs v1.34.7 h1:RxETYGXhRlRxL96mtab1lQ9fPVPIJFXuOI3uRL/MuHI=
-github.com/aws/aws-sdk-go-v2/service/sqs v1.34.7/go.mod h1:zn0Oy7oNni7XIGoAd6bHBTVtX06OrnpvT1kww8jxyi8=
-github.com/aws/aws-sdk-go-v2/service/sso v1.22.6 h1:o++HUDXlbrTl4PSal3YHtdErQxB8mDGAtkKNXBWPfIU=
-github.com/aws/aws-sdk-go-v2/service/sso v1.22.6/go.mod h1:eEygMHnTKH/3kNp9Jr1n3PdejuSNcgwLe1dWgQtO0VQ=
-github.com/aws/aws-sdk-go-v2/service/ssooidc v1.26.6 h1:yCHcQCOwTfIsc8DoEhM3qXPxD+j8CbI6t1K3dNzsWV0=
-github.com/aws/aws-sdk-go-v2/service/ssooidc v1.26.6/go.mod h1:bCbAxKDqNvkHxRaIMnyVPXPo+OaPRwvmgzMxbz1VKSA=
-github.com/aws/aws-sdk-go-v2/service/sts v1.30.6 h1:TrQadF7GcqvQ63kgwEcjlrVc2Fa0wpgLT0xtc73uAd8=
-github.com/aws/aws-sdk-go-v2/service/sts v1.30.6/go.mod h1:NXi1dIAGteSaRLqYgarlhP/Ij0cFT+qmCwiJqWh/U5o=
+github.com/aws/aws-sdk-go-v2/service/sqs v1.34.8 h1:t3TzmBX0lpDNtLhl7vY97VMvLtxp/KTvjjj2X3s6SUQ=
+github.com/aws/aws-sdk-go-v2/service/sqs v1.34.8/go.mod h1:zn0Oy7oNni7XIGoAd6bHBTVtX06OrnpvT1kww8jxyi8=
+github.com/aws/aws-sdk-go-v2/service/sso v1.22.7 h1:pIaGg+08llrP7Q5aiz9ICWbY8cqhTkyy+0SHvfzQpTc=
+github.com/aws/aws-sdk-go-v2/service/sso v1.22.7/go.mod h1:eEygMHnTKH/3kNp9Jr1n3PdejuSNcgwLe1dWgQtO0VQ=
+github.com/aws/aws-sdk-go-v2/service/ssooidc v1.26.7 h1:/Cfdu0XV3mONYKaOt1Gr0k1KvQzkzPyiKUdlWJqy+J4=
+github.com/aws/aws-sdk-go-v2/service/ssooidc v1.26.7/go.mod h1:bCbAxKDqNvkHxRaIMnyVPXPo+OaPRwvmgzMxbz1VKSA=
+github.com/aws/aws-sdk-go-v2/service/sts v1.30.7 h1:NKTa1eqZYw8tiHSRGpP0VtTdub/8KNk8sDkNPFaOKDE=
+github.com/aws/aws-sdk-go-v2/service/sts v1.30.7/go.mod h1:NXi1dIAGteSaRLqYgarlhP/Ij0cFT+qmCwiJqWh/U5o=
 github.com/aws/smithy-go v1.20.4 h1:2HK1zBdPgRbjFOHlfeQZfpC4r72MOb9bZkiFwggKO+4=
 github.com/aws/smithy-go v1.20.4/go.mod h1:irrKGvNn1InZwb2d7fkIRNucdfwR8R+Ts3wxYa/cJHg=
 github.com/aymanbagabas/go-osc52/v2 v2.0.1 h1:HwpRHbFMcZLEVr42D4p7XBqjyuxQH5SMiErDT4WkJ2k=
 github.com/aymanbagabas/go-osc52/v2 v2.0.1/go.mod h1:uYgXzlJ7ZpABp8OJ+exZzJJhRNQ2ASbcXHWsFqH8hp8=
-github.com/charmbracelet/bubbles v0.19.0 h1:gKZkKXPP6GlDk6EcfujDK19PCQqRjaJZQ7QRERx1UF0=
-github.com/charmbracelet/bubbles v0.19.0/go.mod h1:WILteEqZ+krG5c3ntGEMeG99nCupcuIk7V0/zOP0tOA=
-github.com/charmbracelet/bubbletea v1.1.0 h1:FjAl9eAL3HBCHenhz/ZPjkKdScmaS5SK69JAK2YJK9c=
-github.com/charmbracelet/bubbletea v1.1.0/go.mod h1:9Ogk0HrdbHolIKHdjfFpyXJmiCzGwy+FesYkZr7hYU4=
+github.com/charmbracelet/bubbles v0.20.0 h1:jSZu6qD8cRQ6k9OMfR1WlM+ruM8fkPWkHvQWD9LIutE=
+github.com/charmbracelet/bubbles v0.20.0/go.mod h1:39slydyswPy+uVOHZ5x/GjwVAFkCsV8IIVy+4MhzwwU=
+github.com/charmbracelet/bubbletea v1.1.1 h1:KJ2/DnmpfqFtDNVTvYZ6zpPFL9iRCRr0qqKOCvppbPY=
+github.com/charmbracelet/bubbletea v1.1.1/go.mod h1:9Ogk0HrdbHolIKHdjfFpyXJmiCzGwy+FesYkZr7hYU4=
 github.com/charmbracelet/lipgloss v0.13.0 h1:4X3PPeoWEDCMvzDvGmTajSyYPcZM4+y8sCA/SsA3cjw=
 github.com/charmbracelet/lipgloss v0.13.0/go.mod h1:nw4zy0SBX/F/eAO1cWdcvy6qnkDUxr8Lw7dvFrAIbbY=
-github.com/charmbracelet/x/ansi v0.2.3 h1:VfFN0NUpcjBRd4DnKfRaIRo53KRgey/nhOoEqosGDEY=
-github.com/charmbracelet/x/ansi v0.2.3/go.mod h1:dk73KoMTT5AX5BsX0KrqhsTqAnhZZoCBjs7dGWp4Ktw=
+github.com/charmbracelet/x/ansi v0.3.1 h1:CRO6lc/6HCx2/D6S/GZ87jDvRvk6GtPyFP+IljkNtqI=
+github.com/charmbracelet/x/ansi v0.3.1/go.mod h1:dk73KoMTT5AX5BsX0KrqhsTqAnhZZoCBjs7dGWp4Ktw=
 github.com/charmbracelet/x/term v0.2.0 h1:cNB9Ot9q8I711MyZ7myUR5HFWL/lc3OpU8jZ4hwm0x0=
 github.com/charmbracelet/x/term v0.2.0/go.mod h1:GVxgxAbjUrmpvIINHIQnJJKpMlHiZ4cktEQCN6GWyF0=
 github.com/erikgeiser/coninput v0.0.0-20211004153227-1c3628e74d0f h1:Y/CXytFA4m6baUTXGLOoWe4PQhGxaX0KpnayAqC48p4=
 github.com/erikgeiser/coninput v0.0.0-20211004153227-1c3628e74d0f/go.mod h1:vw97MGsxSvLiUE2X8qFplwetxpGLQrlU1Q9AUEIzCaM=
 github.com/kylelemons/godebug v1.1.0 h1:RPNrshWIDI6G2gRW9EHilWtl7Z6Sb1BR0xunSBf0SNc=
 github.com/kylelemons/godebug v1.1.0/go.mod h1:9/0rRGxNHcop5bhtWyNeEfOS8JIWk580+fNqagV/RAw=
 github.com/lucasb-eyer/go-colorful v1.2.0 h1:1nnpGOrhyZZuNyfu1QjKiUICQ74+3FNCN69Aj6K7nkY=
 github.com/lucasb-eyer/go-colorful v1.2.0/go.mod h1:R4dSotOR9KMtayYi1e77YzuveK+i7ruzyGqttikkLy0=
 github.com/mattn/go-isatty v0.0.20 h1:xfD0iDuEKnDkl03q4limB+vH+GxLEtL/jb4xVJSWWEY=
 github.com/mattn/go-isatty v0.0.20/go.mod h1:W+V8PltTTMOvKvAeJH7IuucS94S2C6jfK/D7dTCTo3Y=
@@ -62,14 +62,14 @@ github.com/rivo/uniseg v0.2.0/go.mod h1:J6wj4VEh+S6ZtnVlnTBMWIodfgj8LQOQFoIToxlJ
 github.com/rivo/uniseg v0.4.7 h1:WUdvkW8uEhrYfLC4ZzdpI2ztxP1I582+49Oc5Mq64VQ=
 github.com/rivo/uniseg v0.4.7/go.mod h1:FN3SvrM+Zdj16jyLfmOkMNblXMcoc8DfTHruCPUcx88=
 github.com/sahilm/fuzzy v0.1.1 h1:ceu5RHF8DGgoi+/dR5PsECjCDH1BE3Fnmpo7aVXOdRA=
 github.com/sahilm/fuzzy v0.1.1/go.mod h1:VFvziUEIMCrT6A6tw2RFIXPXXmzXbOsSHF0DOI8ZK9Y=
 github.com/tidwall/pretty v1.2.1 h1:qjsOFOWWQl+N3RsoF5/ssm1pHmJJwhjlSbZ51I6wMl4=
 github.com/tidwall/pretty v1.2.1/go.mod h1:ITEVvHYasfjBbM0u2Pg8T2nJnzm8xPwvNhhsoaGGjNU=
 golang.org/x/sync v0.8.0 h1:3NFvSEYkUoMifnESzZl15y791HH1qU2xm6eCJU5ZPXQ=
 golang.org/x/sync v0.8.0/go.mod h1:Czt+wKu1gCyEFDUtn0jG5QVvpJ6rzVqr5aXyt9drQfk=
 golang.org/x/sys v0.0.0-20210809222454-d867a43fc93e/go.mod h1:oPkhp1MJrh7nUepCBck5+mAzfO9JrbApNNgaTdGDITg=
 golang.org/x/sys v0.6.0/go.mod h1:oPkhp1MJrh7nUepCBck5+mAzfO9JrbApNNgaTdGDITg=
-golang.org/x/sys v0.24.0 h1:Twjiwq9dn6R1fQcyiK+wQyHWfaz/BJB+YIpzU/Cv3Xg=
-golang.org/x/sys v0.24.0/go.mod h1:/VUhepiaJMQUp4+oa/7Zr1D23ma6VTLIYjOOTFZPUcA=
-golang.org/x/text v0.17.0 h1:XtiM5bkSOt+ewxlOE/aE/AKEHibwj/6gvWMl9Rsh0Qc=
-golang.org/x/text v0.17.0/go.mod h1:BuEKDfySbSR4drPmRPG/7iBdf8hvFMuRexcpahXilzY=
+golang.org/x/sys v0.25.0 h1:r+8e+loiHxRqhXVl6ML1nO3l1+oFoWbnlu2Ehimmi34=
+golang.org/x/sys v0.25.0/go.mod h1:/VUhepiaJMQUp4+oa/7Zr1D23ma6VTLIYjOOTFZPUcA=
+golang.org/x/text v0.18.0 h1:XvMDiNzPAl0jr17s6W9lcaIhGUfUORdGCNsuLmPG224=
+golang.org/x/text v0.18.0/go.mod h1:BuEKDfySbSR4drPmRPG/7iBdf8hvFMuRexcpahXilzY=
diff --git a/cueitup.go b/main.go
similarity index 54%
rename from cueitup.go
rename to main.go
index 4a28df2..0480ca2 100644
--- a/cueitup.go
+++ b/main.go
@@ -1,9 +1,14 @@
 package main
 
 import (
+	"os"
+
 	"github.com/dhth/cueitup/cmd"
 )
 
 func main() {
-	cmd.Execute()
+	err := cmd.Execute()
+	if err != nil {
+		os.Exit(1)
+	}
 }
diff --git a/ui/model/cmds.go b/ui/model/cmds.go
index 837bbeb..038dbd3 100644
--- a/ui/model/cmds.go
+++ b/ui/model/cmds.go
@@ -8,91 +8,86 @@ import (
 	"strconv"
 	"strings"
 	"time"
 
 	"github.com/aws/aws-sdk-go-v2/aws"
 	"github.com/aws/aws-sdk-go-v2/service/sqs"
 	"github.com/aws/aws-sdk-go-v2/service/sqs/types"
 	tea "github.com/charmbracelet/bubbletea"
 )
 
-func (m model) FetchMessages(maxMessages int32, waitTime int32) tea.Cmd {
+func (m Model) FetchMessages(maxMessages int32, waitTime int32) tea.Cmd {
 	return func() tea.Msg {
-
 		var messages []types.Message
 		var messagesValues []string
 		var keyValues []string
 		result, err := m.sqsClient.ReceiveMessage(context.TODO(),
 			// WaitTimeSeconds > 0 enables long polling
 			// https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-short-and-long-polling.html#sqs-long-polling
 			&sqs.ReceiveMessageInput{
-				QueueUrl:            aws.String(m.queueUrl),
+				QueueUrl:            aws.String(m.queueURL),
 				MaxNumberOfMessages: maxMessages,
 				WaitTimeSeconds:     waitTime,
 				VisibilityTimeout:   30,
 			})
 		if err != nil {
 			return SQSMsgFetchedMsg{
 				messages: nil,
 				err:      err,
 			}
-		} else {
-			messages = result.Messages
-			for _, message := range messages {
-				msgValue, keyValue, _ := getMessageData(&message, m.msgConsumptionConf)
-				messagesValues = append(messagesValues, msgValue)
-				keyValues = append(keyValues, keyValue)
-			}
+		}
+		messages = result.Messages
+		for _, message := range messages {
+			msgValue, keyValue, _ := getMessageData(&message, m.msgConsumptionConf)
+			messagesValues = append(messagesValues, msgValue)
+			keyValues = append(keyValues, keyValue)
 		}
 
 		return SQSMsgFetchedMsg{
 			messages:      messages,
 			messageValues: messagesValues,
 			keyValues:     keyValues,
 			err:           nil,
 		}
 	}
 }
 
-func DeleteMessages(client *sqs.Client, queueUrl string, messages []types.Message) tea.Cmd {
+func DeleteMessages(client *sqs.Client, queueURL string, messages []types.Message) tea.Cmd {
 	return func() tea.Msg {
-
 		entries := make([]types.DeleteMessageBatchRequestEntry, len(messages))
 		for msgIndex := range messages {
 			entries[msgIndex].Id = aws.String(fmt.Sprintf("%v", msgIndex))
 			entries[msgIndex].ReceiptHandle = messages[msgIndex].ReceiptHandle
 		}
 		_, err := client.DeleteMessageBatch(context.TODO(),
 			&sqs.DeleteMessageBatchInput{
 				Entries:  entries,
-				QueueUrl: aws.String(queueUrl),
+				QueueUrl: aws.String(queueURL),
 			})
 		if err != nil {
 			return SQSMsgsDeletedMsg{
 				err: err,
 			}
 		}
 
 		return SQSMsgsDeletedMsg{}
 	}
 }
 
-func GetQueueMsgCount(client *sqs.Client, queueUrl string) tea.Cmd {
+func GetQueueMsgCount(client *sqs.Client, queueURL string) tea.Cmd {
 	return func() tea.Msg {
-
 		approxMsgCountType := types.QueueAttributeNameApproximateNumberOfMessages
 		attribute, err := client.GetQueueAttributes(context.TODO(),
 			&sqs.GetQueueAttributesInput{
-				QueueUrl:       aws.String(queueUrl),
+				QueueUrl:       aws.String(queueURL),
 				AttributeNames: []types.QueueAttributeName{approxMsgCountType},
 			})
-
 		if err != nil {
 			return QueueMsgCountFetchedMsg{
 				approxMsgCount: -1,
 				err:            err,
 			}
 		}
 
 		countStr := attribute.Attributes[string(approxMsgCountType)]
 		count, err := strconv.Atoi(countStr)
 		if err != nil {
@@ -103,32 +98,32 @@ func GetQueueMsgCount(client *sqs.Client, queueUrl string) tea.Cmd {
 		}
 		return QueueMsgCountFetchedMsg{
 			approxMsgCount: count,
 		}
 	}
 }
 
 func saveRecordValueToDisk(filePath string, msgValue string, msgFmt MsgFmt) tea.Cmd {
 	return func() tea.Msg {
 		dir := filepath.Dir(filePath)
-		err := os.MkdirAll(dir, 0755)
+		err := os.MkdirAll(dir, 0o755)
 		if err != nil {
 			return RecordSavedToDiskMsg{err: err}
 		}
 		var data string
 		switch msgFmt {
-		case JsonFmt:
+		case JSONFmt:
 			data = fmt.Sprintf("json\n%s\n", msgValue)
 		case PlainTxtFmt:
 			data = msgValue
 		}
-		err = os.WriteFile(filePath, []byte(data), 0644)
+		err = os.WriteFile(filePath, []byte(data), 0o644)
 		if err != nil {
 			return RecordSavedToDiskMsg{err: err}
 		}
 		return RecordSavedToDiskMsg{path: filePath}
 	}
 }
 
 func setContextSearchValues(userInput string) tea.Cmd {
 	return func() tea.Msg {
 		valuesEls := strings.Split(userInput, ",")
diff --git a/ui/model/delegate.go b/ui/model/delegate.go
index 80805d4..be199a1 100644
--- a/ui/model/delegate.go
+++ b/ui/model/delegate.go
@@ -11,35 +11,34 @@ func newAppItemDelegate() list.DefaultDelegate {
 	d := list.NewDefaultDelegate()
 
 	d.Styles.SelectedTitle = d.Styles.
 		SelectedTitle.
 		Foreground(lipgloss.Color(listColor)).
 		BorderLeftForeground(lipgloss.Color(listColor))
 	d.Styles.SelectedDesc = d.Styles.
 		SelectedTitle
 
 	d.UpdateFunc = func(msg tea.Msg, m *list.Model) tea.Cmd {
-		switch msgType := msg.(type) {
-		case tea.KeyMsg:
-			switch {
-			case key.Matches(msgType,
-				list.DefaultKeyMap().CursorUp,
-				list.DefaultKeyMap().CursorDown,
-				list.DefaultKeyMap().GoToStart,
-				list.DefaultKeyMap().GoToEnd,
-				list.DefaultKeyMap().NextPage,
-				list.DefaultKeyMap().PrevPage,
-			):
-				selected := m.SelectedItem()
-				if selected == nil {
-					return nil
-				}
-				key := selected.FilterValue()
-				return showItemDetails(key)
+		keyMsg, keyMsgOK := msg.(tea.KeyMsg)
+		if !keyMsgOK {
+			return nil
+		}
+		if key.Matches(keyMsg,
+			list.DefaultKeyMap().CursorUp,
+			list.DefaultKeyMap().CursorDown,
+			list.DefaultKeyMap().GoToStart,
+			list.DefaultKeyMap().GoToEnd,
+			list.DefaultKeyMap().NextPage,
+			list.DefaultKeyMap().PrevPage,
+		) {
+			selected := m.SelectedItem()
+			if selected == nil {
+				return nil
 			}
-
+			key := selected.FilterValue()
+			return showItemDetails(key)
 		}
 		return nil
 	}
 
 	return d
 }
diff --git a/ui/model/help.go b/ui/model/help.go
index 76c4bc5..7f9c7d6 100644
--- a/ui/model/help.go
+++ b/ui/model/help.go
@@ -1,61 +1,59 @@
 package model
 
 import "fmt"
 
-var (
-	HelpText = fmt.Sprintf(`
+var HelpText = fmt.Sprintf(`
   %s
 %s
   %s
 
   %s
 %s
   %s
 %s
   %s
 %s
 `,
-		helpHeaderStyle.Render("cueitup Reference Manual"),
-		helpSectionStyle.Render(`
+	helpHeaderStyle.Render("cueitup Reference Manual"),
+	helpSectionStyle.Render(`
   (scroll line by line with j/k/arrow keys or by half a page with <c-d>/<c-u>)
 
   cueitup has 3 views:
   - Message List View
   - Message Value View
   - Help View (this one)
 `),
-		helpHeaderStyle.Render("Keyboard Shortcuts"),
-		helpHeaderStyle.Render("General"),
-		helpSectionStyle.Render(`
+	helpHeaderStyle.Render("Keyboard Shortcuts"),
+	helpHeaderStyle.Render("General"),
+	helpSectionStyle.Render(`
       <tab>                          Switch focus to next section
       <s-tab>                        Switch focus to previous section
       1                              Maximize message value view
       ?                              Show help view
 `),
-		helpHeaderStyle.Render("Message List View"),
-		helpSectionStyle.Render(`
+	helpHeaderStyle.Render("Message List View"),
+	helpSectionStyle.Render(`
       h/<Up>                         Move cursor up
       k/<Down>                       Move cursor down
       n                              Fetch the next message from the queue
       N                              Fetch up to 10 more messages from the queue
       }                              Fetch up to 100 more messages from the queue
       d                              Toggle deletion mode; cueitup will delete messages
                                          after reading them
       <ctrl+s>                       Toggle contextual search prompt
       <ctrl+f>                       Toggle contextual filtering ON/OFF
       <ctrl+p>                       Toggle queue message count polling ON/OFF; ON by default
       p                              Toggle persist mode (cueitup will start persisting
                                          messages, at the location
                                          messages/<topic-name>/<timestamp-when-cueitup-started>/<unix-epoch>-<message-id>.md
       s                              Toggle skipping mode; cueitup will consume messages,
                                          but not populate its internal list, effectively
                                          skipping over them
 `),
-		helpHeaderStyle.Render("Message Value View   "),
-		helpSectionStyle.Render(`
+	helpHeaderStyle.Render("Message Value View   "),
+	helpSectionStyle.Render(`
       q                              Minimize section, and return focus to list view
       [,h                            Show details for the previous entry in the list
       ],l                            Show details for the next entry in the list
 `),
-	)
 )
diff --git a/ui/model/initial.go b/ui/model/initial.go
index e037600..28dd396 100644
--- a/ui/model/initial.go
+++ b/ui/model/initial.go
@@ -5,45 +5,44 @@ import (
 	"os"
 	"strings"
 	"time"
 
 	"github.com/aws/aws-sdk-go-v2/service/sqs"
 	"github.com/charmbracelet/bubbles/list"
 	"github.com/charmbracelet/bubbles/textinput"
 	"github.com/charmbracelet/lipgloss"
 )
 
-func InitialModel(sqsClient *sqs.Client, queueUrl string, msgConsumptionConf MsgConsumptionConf) model {
-
+func InitialModel(sqsClient *sqs.Client, queueURL string, msgConsumptionConf MsgConsumptionConf) Model {
 	appDelegate := newAppItemDelegate()
 	jobItems := make([]list.Item, 0)
 
-	queueParts := strings.Split(queueUrl, "/")
+	queueParts := strings.Split(queueURL, "/")
 	queueName := queueParts[len(queueParts)-1]
 	currentTime := time.Now()
 	timeString := currentTime.Format("2006-01-02-15-04-05")
 	persistDir := fmt.Sprintf("messages/%s/%s", queueName, timeString)
 
 	ti := textinput.New()
 	ti.Prompt = fmt.Sprintf("Filter messages where %s in > ", msgConsumptionConf.ContextKey)
 	ti.Focus()
 	ti.CharLimit = 100
 	ti.Width = 100
 
 	var dbg bool
 	if len(os.Getenv("DEBUG")) > 0 {
 		dbg = true
 	}
 
-	m := model{
+	m := Model{
 		sqsClient:            sqsClient,
-		queueUrl:             queueUrl,
+		queueURL:             queueURL,
 		msgConsumptionConf:   msgConsumptionConf,
 		pollForQueueMsgCount: true,
 		msgsList:             list.New(jobItems, appDelegate, listWidth+10, 0),
 		recordValueStore:     make(map[string]string),
 		persistDir:           persistDir,
 		contextSearchInput:   ti,
 		showHelpIndicator:    true,
 		debugMode:            dbg,
 		firstFetch:           true,
 	}
diff --git a/ui/model/model.go b/ui/model/model.go
index 73f2a00..cce4fc8 100644
--- a/ui/model/model.go
+++ b/ui/model/model.go
@@ -15,36 +15,36 @@ type stateView uint
 const (
 	msgsListView stateView = iota
 	msgValueView
 	helpView
 	contextualSearchView
 )
 
 type MsgFmt uint
 
 const (
-	JsonFmt MsgFmt = iota
+	JSONFmt MsgFmt = iota
 	PlainTxtFmt
 )
 
 const msgCountTickInterval = time.Second * 3
 
 type MsgConsumptionConf struct {
 	Format     MsgFmt
 	SubsetKey  string
 	ContextKey string
 }
 
-type model struct {
+type Model struct {
 	deserializationFmt   MsgFmt
 	sqsClient            *sqs.Client
-	queueUrl             string
+	queueURL             string
 	msgConsumptionConf   MsgConsumptionConf
 	activeView           stateView
 	lastView             stateView
 	pollForQueueMsgCount bool
 	msgsList             list.Model
 	helpVP               viewport.Model
 	showHelpIndicator    bool
 	msgValueVP           viewport.Model
 	recordValueStore     map[string]string
 	contextSearchInput   textinput.Model
@@ -58,17 +58,17 @@ type model struct {
 	helpVPReady          bool
 	vpFullScreen         bool
 	terminalWidth        int
 	terminalHeight       int
 	message              string
 	errorMsg             string
 	debugMode            bool
 	firstFetch           bool
 }
 
-func (m model) Init() tea.Cmd {
+func (m Model) Init() tea.Cmd {
 	return tea.Batch(
-		GetQueueMsgCount(m.sqsClient, m.queueUrl),
+		GetQueueMsgCount(m.sqsClient, m.queueURL),
 		tickEvery(msgCountTickInterval),
 		hideHelp(time.Minute*1),
 	)
 }
diff --git a/ui/model/msgs.go b/ui/model/msgs.go
index 15ec498..e1e52a4 100644
--- a/ui/model/msgs.go
+++ b/ui/model/msgs.go
@@ -1,18 +1,20 @@
 package model
 
 import (
 	"github.com/aws/aws-sdk-go-v2/service/sqs/types"
 )
 
-type MsgCountTickMsg struct{}
-type HideHelpMsg struct{}
+type (
+	MsgCountTickMsg struct{}
+	HideHelpMsg     struct{}
+)
 
 type SQSMsgFetchedMsg struct {
 	messages      []types.Message
 	messageValues []string
 	keyValues     []string
 	err           error
 }
 
 type QueueMsgCountFetchedMsg struct {
 	approxMsgCount int
diff --git a/ui/model/types.go b/ui/model/types.go
index 565b8cb..48c5646 100644
--- a/ui/model/types.go
+++ b/ui/model/types.go
@@ -18,12 +18,12 @@ func (item msgItem) Title() string {
 }
 
 func (item msgItem) Description() string {
 	if item.contextKeyValue != "" {
 		return RightPadTrim(fmt.Sprintf("%s: %s", RightPadTrim(item.contextKeyName, 10), item.contextKeyValue), listWidth)
 	}
 	return ""
 }
 
 func (item msgItem) FilterValue() string {
-	return string(*item.message.MessageId)
+	return *item.message.MessageId
 }
diff --git a/ui/model/update.go b/ui/model/update.go
index ba99b5b..41ae3b9 100644
--- a/ui/model/update.go
+++ b/ui/model/update.go
@@ -3,23 +3,26 @@ package model
 import (
 	"fmt"
 	"time"
 
 	"github.com/charmbracelet/bubbles/list"
 	"github.com/charmbracelet/bubbles/viewport"
 	tea "github.com/charmbracelet/bubbletea"
 	"github.com/tidwall/pretty"
 )
 
-const useHighPerformanceRenderer = false
+const (
+	useHighPerformanceRenderer = false
+	fetchingIndicator          = " ..."
+)
 
-func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
+func (m Model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
 	var cmds []tea.Cmd
 	m.message = ""
 	m.errorMsg = ""
 
 	switch msg := msg.(type) {
 	case tea.KeyMsg:
 		switch msg.String() {
 		case "ctrl+c", "q":
 			switch m.activeView {
 			case msgsListView:
@@ -41,31 +44,31 @@ func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
 		case "enter":
 			if m.activeView == contextualSearchView {
 				m.activeView = m.lastView
 				if len(m.contextSearchInput.Value()) > 0 {
 					cmds = append(cmds, setContextSearchValues(m.contextSearchInput.Value()))
 				} else {
 					m.filterMessages = false
 				}
 			}
 		case "n", " ":
-			m.message = " ..."
+			m.message = fetchingIndicator
 			cmds = append(cmds, m.FetchMessages(1, 0))
 		case "N":
-			m.message = " ..."
+			m.message = fetchingIndicator
 			for i := 0; i < 10; i++ {
 				cmds = append(cmds,
 					m.FetchMessages(1, 0),
 				)
 			}
 		case "}":
-			m.message = " ..."
+			m.message = fetchingIndicator
 			for i := 0; i < 20; i++ {
 				cmds = append(cmds,
 					m.FetchMessages(5, 0),
 				)
 			}
 		case "?":
 			m.lastView = m.activeView
 			m.activeView = helpView
 		case "d":
 			if m.activeView == msgsListView {
@@ -96,21 +99,21 @@ func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
 				selected := m.msgsList.SelectedItem()
 				if selected != nil {
 					result := string(pretty.Color([]byte(m.recordValueStore[selected.FilterValue()]), nil))
 					m.msgValueVP.SetContent(result)
 				}
 			}
 		case "ctrl+p":
 			m.pollForQueueMsgCount = !m.pollForQueueMsgCount
 			if m.pollForQueueMsgCount {
 				cmds = append(cmds,
-					tea.Batch(GetQueueMsgCount(m.sqsClient, m.queueUrl),
+					tea.Batch(GetQueueMsgCount(m.sqsClient, m.queueURL),
 						tickEvery(msgCountTickInterval),
 					),
 				)
 			}
 		case "ctrl+s":
 			if m.activeView == msgsListView {
 				m.lastView = m.activeView
 				m.activeView = contextualSearchView
 			}
 		case "ctrl+f":
@@ -184,82 +187,84 @@ func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
 		m.contextSearchInput.SetValue("")
 		m.filterMessages = true
 
 	case HideHelpMsg:
 		m.showHelpIndicator = false
 
 	case SQSMsgFetchedMsg:
 		if msg.err != nil {
 			m.errorMsg = msg.err.Error()
 		} else {
-			switch m.skipRecords {
-			case false:
+			if !m.skipRecords {
 				vPresenceMap := make(map[string]bool)
 				if m.filterMessages && len(m.contextSearchValues) > 0 {
 					for _, p := range m.contextSearchValues {
 						vPresenceMap[p] = true
 					}
 				}
 				for i, message := range msg.messages {
-
 					// only save/persist values that are requested to be filtered
 					if m.filterMessages && !(msg.keyValues[i] != "" && vPresenceMap[msg.keyValues[i]]) {
 						continue
 					}
 
 					m.msgsList.InsertItem(len(m.msgsList.Items()),
-						msgItem{message: message,
+						msgItem{
+							message:         message,
 							messageValue:    msg.messageValues[i],
 							contextKeyName:  m.msgConsumptionConf.ContextKey,
 							contextKeyValue: msg.keyValues[i],
 						},
 					)
 					m.recordValueStore[*message.MessageId] = msg.messageValues[i]
+
 					if m.persistRecords {
 						prefix := time.Now().Unix()
 						filePath := fmt.Sprintf("%s/%d-%s.md", m.persistDir, prefix, *message.MessageId)
 						cmds = append(cmds,
 							saveRecordValueToDisk(
 								filePath,
 								msg.messageValues[i],
 								m.msgConsumptionConf.Format,
 							),
 						)
 					}
 				}
+
 				if m.deleteMsgs {
 					cmds = append(cmds,
 						DeleteMessages(m.sqsClient,
-							m.queueUrl,
+							m.queueURL,
 							msg.messages),
 					)
 				}
+
 				if m.firstFetch {
 					selected := m.msgsList.SelectedItem()
 					if selected != nil {
 						result := string(pretty.Color([]byte(m.recordValueStore[selected.FilterValue()]), nil))
 						m.msgValueVP.SetContent(result)
 						m.firstFetch = false
 					}
 				}
 			}
 		}
 	case KMsgChosenMsg:
 		switch m.deserializationFmt {
-		case JsonFmt:
+		case JSONFmt:
 			result := string(pretty.Color([]byte(m.recordValueStore[msg.key]), nil))
 			m.msgValueVP.SetContent(result)
 		default:
 			m.msgValueVP.SetContent(m.recordValueStore[msg.key])
 		}
 	case MsgCountTickMsg:
-		cmds = append(cmds, GetQueueMsgCount(m.sqsClient, m.queueUrl))
+		cmds = append(cmds, GetQueueMsgCount(m.sqsClient, m.queueURL))
 		if m.pollForQueueMsgCount {
 			cmds = append(cmds, tickEvery(msgCountTickInterval))
 		}
 	case QueueMsgCountFetchedMsg:
 		if msg.err != nil {
 			m.errorMsg = msg.err.Error()
 		} else {
 			m.msgsList.Title = fmt.Sprintf("Messages (%d in queue)", msg.approxMsgCount)
 		}
 	}
diff --git a/ui/model/utils.go b/ui/model/utils.go
index 21bdc83..24bb3d3 100644
--- a/ui/model/utils.go
+++ b/ui/model/utils.go
@@ -80,30 +80,29 @@ func getRecordValueJSONNested(message *types.Message, extractKey string, context
 	}
 
 	return string(nestedBytes), contextualValue.(string), nil
 }
 
 func getMessageData(message *types.Message, msgConsumptionConf MsgConsumptionConf) (string, string, error) {
 	var msgValue, keyValue string
 	var err error
 
 	switch msgConsumptionConf.Format {
-	case JsonFmt:
+	case JSONFmt:
 		if msgConsumptionConf.SubsetKey != "" {
 			msgValue,
 				keyValue,
 				err = getRecordValueJSONNested(message,
 				msgConsumptionConf.SubsetKey,
 				msgConsumptionConf.ContextKey,
 			)
 		} else {
 			msgValue, err = getRecordValueJSONFull(message)
 		}
 	case PlainTxtFmt:
 		msgValue = *message.Body
 	}
 	if err != nil {
 		return "", "", err
-	} else {
-		return msgValue, keyValue, nil
 	}
+	return msgValue, keyValue, nil
 }
diff --git a/ui/model/view.go b/ui/model/view.go
index 5de1437..5e2bf63 100644
--- a/ui/model/view.go
+++ b/ui/model/view.go
@@ -1,23 +1,21 @@
 package model
 
 import (
 	"fmt"
 
 	"github.com/charmbracelet/lipgloss"
 )
 
-var (
-	listWidth = 50
-)
+var listWidth = 50
 
-func (m model) View() string {
+func (m Model) View() string {
 	var content string
 	var footer string
 	var mode string
 	var statusBar string
 	var debugMsg string
 	var msgValVPTitleStyle lipgloss.Style
 
 	if m.message != "" {
 		statusBar = Trim(m.message, 120)
 	}
diff --git a/ui/ui.go b/ui/ui.go
index f13c6e7..381c448 100644
--- a/ui/ui.go
+++ b/ui/ui.go
@@ -1,27 +1,26 @@
 package ui
 
 import (
+	"errors"
 	"fmt"
-	"log"
 	"os"
 
 	"github.com/aws/aws-sdk-go-v2/service/sqs"
 	tea "github.com/charmbracelet/bubbletea"
 	"github.com/dhth/cueitup/ui/model"
 )
 
-func RenderUI(sqsClient *sqs.Client, queueUrl string, msgConsumptionConf model.MsgConsumptionConf) {
+var errFailedToConfigureDebugging = errors.New("failed to configure debugging")
 
+func RenderUI(sqsClient *sqs.Client, queueURL string, msgConsumptionConf model.MsgConsumptionConf) error {
 	if len(os.Getenv("DEBUG")) > 0 {
 		f, err := tea.LogToFile("debug.log", "debug")
 		if err != nil {
-			fmt.Println("fatal:", err)
-			os.Exit(1)
+			return fmt.Errorf("%w: %s", errFailedToConfigureDebugging, err.Error())
 		}
 		defer f.Close()
 	}
-	p := tea.NewProgram(model.InitialModel(sqsClient, queueUrl, msgConsumptionConf), tea.WithAltScreen())
-	if _, err := p.Run(); err != nil {
-		log.Fatalf("Something went wrong %s", err)
-	}
+	p := tea.NewProgram(model.InitialModel(sqsClient, queueURL, msgConsumptionConf), tea.WithAltScreen())
+	_, err := p.Run()
+	return err
 }
diff --git a/yamlfmt.yml b/yamlfmt.yml
new file mode 100644
index 0000000..9d3236a
--- /dev/null
+++ b/yamlfmt.yml
@@ -0,0 +1,2 @@
+formatter:
+  retain_line_breaks_single: true
```

</details>

👉 The "distilled-diff" version of the same diff looks like the following:

<details><summary> expand </summary>

```diff
diff --git a/5c52f68/cmd/root.go b/11a0436/cmd/root.go
index 7e245d5..a1c36f6 100644
--- a/5c52f68/cmd/root.go
+++ b/11a0436/cmd/root.go
@@ -1,2 +1 @@
-func die(msg string, args ...any)
-func Execute()
+func Execute() error
diff --git a/5c52f68/cueitup.go b/11a0436/main.go
similarity index 100%
rename from 5c52f68/cueitup.go
rename to 11a0436/main.go
diff --git a/5c52f68/ui/model/cmds.go b/11a0436/ui/model/cmds.go
index 35aaf4d..e429709 100644
--- a/5c52f68/ui/model/cmds.go
+++ b/11a0436/ui/model/cmds.go
@@ -1,2 +1,2 @@
-func DeleteMessages(client *sqs.Client, queueUrl string, messages []types.Message) tea.Cmd
-func GetQueueMsgCount(client *sqs.Client, queueUrl string) tea.Cmd
+func DeleteMessages(client *sqs.Client, queueURL string, messages []types.Message) tea.Cmd
+func GetQueueMsgCount(client *sqs.Client, queueURL string) tea.Cmd
@@ -8 +8 @@ func hideHelp(interval time.Duration) tea.Cmd
-func (m model) FetchMessages(maxMessages int32, waitTime int32) tea.Cmd
+func (m Model) FetchMessages(maxMessages int32, waitTime int32) tea.Cmd
diff --git a/5c52f68/ui/model/initial.go b/11a0436/ui/model/initial.go
index 1381c76..7654528 100644
--- a/5c52f68/ui/model/initial.go
+++ b/11a0436/ui/model/initial.go
@@ -1 +1 @@
-func InitialModel(sqsClient *sqs.Client, queueUrl string, msgConsumptionConf MsgConsumptionConf) model
+func InitialModel(sqsClient *sqs.Client, queueURL string, msgConsumptionConf MsgConsumptionConf) Model
diff --git a/5c52f68/ui/model/model.go b/11a0436/ui/model/model.go
index 30e48cb..992323e 100644
--- a/5c52f68/ui/model/model.go
+++ b/11a0436/ui/model/model.go
@@ -8 +8 @@ type MsgConsumptionConf struct {
-type model struct {
+type Model struct {
@@ -11 +11 @@ type model struct {
-	queueUrl             string
+	queueURL             string
@@ -38 +38 @@ type model struct {
-func (m model) Init() tea.Cmd
+func (m Model) Init() tea.Cmd
diff --git a/5c52f68/ui/model/msgs.go b/11a0436/ui/model/msgs.go
index 38d7c0c..48a036f 100644
--- a/5c52f68/ui/model/msgs.go
+++ b/11a0436/ui/model/msgs.go
@@ -1,2 +1,4 @@
-type MsgCountTickMsg struct{}
-type HideHelpMsg struct{}
+type (
+	MsgCountTickMsg struct{}
+	HideHelpMsg     struct{}
+)
diff --git a/5c52f68/ui/model/update.go b/11a0436/ui/model/update.go
index 09d4d9e..63ebf89 100644
--- a/5c52f68/ui/model/update.go
+++ b/11a0436/ui/model/update.go
@@ -1 +1 @@
-func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd)
+func (m Model) Update(msg tea.Msg) (tea.Model, tea.Cmd)
diff --git a/5c52f68/ui/model/view.go b/11a0436/ui/model/view.go
index c62ab48..80c398e 100644
--- a/5c52f68/ui/model/view.go
+++ b/11a0436/ui/model/view.go
@@ -1 +1 @@
-func (m model) View() string
+func (m Model) View() string
diff --git a/5c52f68/ui/ui.go b/11a0436/ui/ui.go
index f8e0ecf..ebdc12a 100644
--- a/5c52f68/ui/ui.go
+++ b/11a0436/ui/ui.go
@@ -1 +1 @@
-func RenderUI(sqsClient *sqs.Client, queueUrl string, msgConsumptionConf model.MsgConsumptionConf)
+func RenderUI(sqsClient *sqs.Client, queueURL string, msgConsumptionConf model.MsgConsumptionConf) error
```

</details>

As seen above, the "distilled" version only shows changes in function
signatures, and type definitions, which makes maks reviewing large-scale
refactors easier.

⚡️ Usage
---

```yaml
name: dstlled-diff

on:
  pull_request:

jobs:
  dstlled-diff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - id: get-dstlled-diff
        uses: dhth/dstlled-diff-action@v0.1.0
        with:
          starting-commit: ${{ github.event.pull_request.base.sha }}
          ending-commit: ${{ github.event.pull_request.head.sha }}
          post-comment-on-pr: 'true'
```

The "distilled" diff will be available as an output of the action, via
`steps.get-dstlled-diff.outputs.diff`.

🔡 Inputs
---

Following inputs can be used as `step.with` keys:

| Name                 | Type   | Default | Description                                                        |
|----------------------|--------|---------|--------------------------------------------------------------------|
| `starting-commit`    | String |         | Starting commit for the git revision range                         |
| `ending-commit`      | String |         | Ending commit for the git revision range                           |
| `pattern`            | String | `*`     | Pattern to run dstlled-diff on                                     |
| `directory`          | String | `.`     | Working directory (below repository root)                          |
| `post-comment-on-pr` | Bool   | `false` | Post comment containing dstlled-diff to corresponding pull request |
| `save-diff-to-file`  | Bool   | `false` | Save diff to a local file called `diff.patch`                      |

⚙️ Other use cases
---

### Web Interface

The output of `dstlled-diff` can be rendered in a web view (see it running
[here][2]), as seen in the image below. Code for this can be found
[here](./.github/workflows/web-demo.yml).

![web-demo](https://tools.dhruvs.space/images/dstlled-diff/dstlled-diff-2.png)

[1]: https://github.com/dhth/dstll
[2]: https://dhth.github.io/dstlled-diff-action/
