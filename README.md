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
diff --git a/.github/scripts/get-yamlfmt.sh b/.github/scripts/get-yamlfmt.sh
new file mode 100755
index 0000000..24f7d7c
--- /dev/null
+++ b/.github/scripts/get-yamlfmt.sh
@@ -0,0 +1,35 @@
+#!/usr/bin/env bash
+
+set -e
+
+if [ $# -ne 3 ]; then
+    echo "Usage: $0 <os> <arch> <version>"
+    echo "eg: $0 Linux x86_64 0.13.0"
+    exit 1
+fi
+
+OS="$1"
+ARCH="$2"
+VERSION="$3"
+
+cwd=$(pwd)
+
+temp_dir=$(mktemp -d)
+if [ ! -e ${temp_dir} ]; then
+    echo "Failed to create temporary directory."
+    exit 1
+fi
+
+cd $temp_dir
+
+curl -sSLO "https://github.com/google/yamlfmt/releases/download/v${VERSION}/yamlfmt_${VERSION}_${OS}_${ARCH}.tar.gz"
+curl -sSLO "https://github.com/google/yamlfmt/releases/download/v${VERSION}/checksums.txt"
+
+sha256sum --ignore-missing -c checksums.txt
+
+tar -xzf "yamlfmt_${VERSION}_${OS}_${ARCH}.tar.gz" -C ${temp_dir}/
+cd $cwd
+
+cp "${temp_dir}/yamlfmt" .
+
+rm -r ${temp_dir}
diff --git a/.github/workflows/back-compat-pr.yml b/.github/workflows/back-compat-pr.yml
index b3c8a65..95be6c0 100644
--- a/.github/workflows/back-compat-pr.yml
+++ b/.github/workflows/back-compat-pr.yml
@@ -14,36 +14,38 @@ env:
   GO_VERSION: '1.23.0'
 
 jobs:
   check-back-compat:
     name: build
     strategy:
       matrix:
         os: [ubuntu-latest, macos-latest]
     runs-on: ${{ matrix.os }}
     steps:
-    - name: Set up Go
-      uses: actions/setup-go@v5
-      with:
-        go-version: ${{ env.GO_VERSION }}
-    - uses: actions/checkout@v4
-      with:
-        ref: main
-    - name: build main
-      run: |
-        go build -o hours_main
-        cp hours_main /var/tmp
-        rm hours_main
-    - uses: actions/checkout@v4
-    - name: build head
-      run: |
-        go build -o hours_head
-        cp hours_head /var/tmp
-        rm hours_head
-    - name: Run last version
-      run: |
-        /var/tmp/hours_main --dbpath=/var/tmp/throwaway-1.db report 3d -p
-        /var/tmp/hours_main --dbpath=/var/tmp/throwaway-2.db gen -y
-    - name: Run current version
-      run: |
-        /var/tmp/hours_head --dbpath=/var/tmp/throwaway-1.db report 3d -p
-        /var/tmp/hours_head --dbpath=/var/tmp/throwaway-2.db report 3d -p
+      - name: Set up Go
+        uses: actions/setup-go@v5
+        with:
+          go-version: ${{ env.GO_VERSION }}
+      - uses: actions/checkout@v4
+        with:
+          ref: main
+      - name: build main
+        run: |
+          go build -o hours_main
+          cp hours_main /var/tmp
+          rm hours_main
+      - uses: actions/checkout@v4
+      - name: build head
+        run: |
+          go build -o hours_head
+          cp hours_head /var/tmp
+          rm hours_head
+      - name: Run last version
+        run: |
+          /var/tmp/hours_main --dbpath=/var/tmp/throwaway-1.db report 3d -p
+          /var/tmp/hours_main --dbpath=/var/tmp/throwaway-2.db gen -y
+      - name: Run current version
+        run: |
+          /var/tmp/hours_head --dbpath=/var/tmp/throwaway-1.db report 3d -p
+          /var/tmp/hours_head --dbpath=/var/tmp/throwaway-2.db report 3d -p
+          /var/tmp/hours_head --dbpath=/var/tmp/throwaway-2.db log 3d -p
+          /var/tmp/hours_head --dbpath=/var/tmp/throwaway-2.db stats 3d -p
diff --git a/.github/workflows/back-compat.yml b/.github/workflows/back-compat.yml
index e0c3202..9d35fed 100644
--- a/.github/workflows/back-compat.yml
+++ b/.github/workflows/back-compat.yml
@@ -1,47 +1,49 @@
 name: back-compat
 
 on:
   push:
-    branches: [ "main" ]
+    branches: ["main"]
 
 permissions:
   contents: read
 
 env:
   GO_VERSION: '1.23.0'
 
 jobs:
   check-back-compat:
     name: build
     strategy:
       matrix:
         os: [ubuntu-latest, macos-latest]
     runs-on: ${{ matrix.os }}
     steps:
-    - name: Set up Go
-      uses: actions/setup-go@v5
-      with:
-        go-version: ${{ env.GO_VERSION }}
-    - uses: actions/checkout@v4
-      with:
-        fetch-depth: 2
-    - run: git checkout HEAD~1
-    - name: build last commit
-      run: |
-        go build -o hours_prev
-        cp hours_prev /var/tmp
-        rm hours_prev
-    - run: git checkout main
-    - name: build head
-      run: |
-        go build -o hours_head
-        cp hours_head /var/tmp
-        rm hours_head
-    - name: Run last version
-      run: |
-        /var/tmp/hours_prev --dbpath=/var/tmp/throwaway-1.db report 3d -p
-        /var/tmp/hours_prev --dbpath=/var/tmp/throwaway-2.db gen -y
-    - name: Run current version
-      run: |
-        /var/tmp/hours_head --dbpath=/var/tmp/throwaway-1.db report 3d -p
-        /var/tmp/hours_head --dbpath=/var/tmp/throwaway-2.db report 3d -p
+      - name: Set up Go
+        uses: actions/setup-go@v5
+        with:
+          go-version: ${{ env.GO_VERSION }}
+      - uses: actions/checkout@v4
+        with:
+          fetch-depth: 2
+      - run: git checkout HEAD~1
+      - name: build last commit
+        run: |
+          go build -o hours_prev
+          cp hours_prev /var/tmp
+          rm hours_prev
+      - run: git checkout main
+      - name: build head
+        run: |
+          go build -o hours_head
+          cp hours_head /var/tmp
+          rm hours_head
+      - name: Run last version
+        run: |
+          /var/tmp/hours_prev --dbpath=/var/tmp/throwaway-1.db report 3d -p
+          /var/tmp/hours_prev --dbpath=/var/tmp/throwaway-2.db gen -y
+      - name: Run current version
+        run: |
+          /var/tmp/hours_head --dbpath=/var/tmp/throwaway-1.db report 3d -p
+          /var/tmp/hours_head --dbpath=/var/tmp/throwaway-2.db report 3d -p
+          /var/tmp/hours_head --dbpath=/var/tmp/throwaway-2.db log 3d -p
+          /var/tmp/hours_head --dbpath=/var/tmp/throwaway-2.db stats 3d -p
diff --git a/.github/workflows/build.yml b/.github/workflows/build.yml
index 5b4df82..c5ee333 100644
--- a/.github/workflows/build.yml
+++ b/.github/workflows/build.yml
@@ -1,40 +1,42 @@
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
   GO_VERSION: '1.23.0'
 
 jobs:
   build:
     name: build
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
-    - name: go test
-      run: go test -v ./...
-    - name: run hours
-      run: |
-        go build .
-        ./hours --dbpath=/var/tmp/throwaway-1.db report 3d -p
-        ./hours --dbpath=/var/tmp/throwaway-2.db gen -y
-        ./hours --dbpath=/var/tmp/throwaway-2.db report 3d -p
+      - uses: actions/checkout@v4
+      - name: Set up Go
+        uses: actions/setup-go@v5
+        with:
+          go-version: ${{ env.GO_VERSION }}
+      - name: go build
+        run: go build -v ./...
+      - name: go test
+        run: go test -v ./...
+      - name: run hours
+        run: |
+          go build .
+          ./hours --dbpath=/var/tmp/throwaway-1.db report 3d -p
+          ./hours --dbpath=/var/tmp/throwaway-2.db gen -y
+          ./hours --dbpath=/var/tmp/throwaway-2.db report 3d -p
+          ./hours --dbpath=/var/tmp/throwaway-2.db log 3d -p
+          ./hours --dbpath=/var/tmp/throwaway-2.db stats 3d -p
diff --git a/.github/workflows/lint-yml.yml b/.github/workflows/lint-yml.yml
new file mode 100644
index 0000000..0b98292
--- /dev/null
+++ b/.github/workflows/lint-yml.yml
@@ -0,0 +1,22 @@
+name: lint-yml
+
+on:
+  push:
+    branches: ["main"]
+    paths:
+      - "**.yml"
+  pull_request:
+    paths:
+      - "**.yml"
+
+jobs:
+  lint-yml:
+    runs-on: ubuntu-latest
+    steps:
+      - uses: actions/checkout@v4
+      - name: Get yamlfmt
+        run: |
+          LATEST_VERSION=$(curl -s https://api.github.com/repos/google/yamlfmt/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/' | sed 's/^v//')
+          ./.github/scripts/get-yamlfmt.sh "Linux" "x86_64" "$LATEST_VERSION"
+      - name: Run yamlfmt
+        run: ./yamlfmt -lint -quiet $(find . -name '*.yml')
diff --git a/.github/workflows/lint.yml b/.github/workflows/lint.yml
index ea243b9..a13440b 100644
--- a/.github/workflows/lint.yml
+++ b/.github/workflows/lint.yml
@@ -1,31 +1,31 @@
 name: lint
 
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
   GO_VERSION: '1.23.0'
 
 jobs:
   lint:
     name: lint
     runs-on: ubuntu-latest
     steps:
-    - uses: actions/checkout@v4
-    - name: Set up Go
-      uses: actions/setup-go@v5
-      with:
-        go-version: ${{ env.GO_VERSION }}
-    - name: golangci-lint
-      uses: golangci/golangci-lint-action@v6
-      with:
-        version: v1.60
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
index 0d3a163..b40df19 100644
--- a/.github/workflows/release.yml
+++ b/.github/workflows/release.yml
@@ -5,38 +5,38 @@ on:
     tags:
       - 'v*'
 
 env:
   GO_VERSION: '1.23.0'
 
 jobs:
   release:
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
-    - name: Test
-      run: go test -v ./...
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
+      - name: Store Cosign private key in a file
+        run: 'echo "$COSIGN_KEY" > cosign.key'
+        shell: bash
+        env:
+          COSIGN_KEY: ${{secrets.COSIGN_KEY}}
+      - name: Release Binaries
+        uses: goreleaser/goreleaser-action@v6
+        with:
+          version: latest
+          args: release --clean
+        env:
+          GITHUB_TOKEN: ${{secrets.GH_PAT}}
+          COSIGN_PASSWORD: ${{secrets.COSIGN_PASSWORD}}
diff --git a/.github/workflows/vulncheck.yml b/.github/workflows/vulncheck.yml
index 4720fbc..f857235 100644
--- a/.github/workflows/vulncheck.yml
+++ b/.github/workflows/vulncheck.yml
@@ -1,31 +1,31 @@
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
   GO_VERSION: '1.23.0'
 
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
index a4172f7..173f848 100644
--- a/.golangci.yml
+++ b/.golangci.yml
@@ -1,9 +1,60 @@
 linters:
   enable:
     - errcheck
+    - errname
+    - errorlint
+    - goconst
     - gofumpt
     - gosimple
     - govet
     - ineffassign
+    - nilerr
+    - prealloc
+    - predeclared
+    - revive
+    - rowserrcheck
+    - sqlclosecheck
     - staticcheck
+    - testifylint
+    - thelper
+    - unconvert
     - unused
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
+        arguments: ["fmt.Print", "fmt.Printf", "fmt.Fprintf", "fmt.Fprint"]
diff --git a/.goreleaser.yaml b/.goreleaser.yml
similarity index 95%
rename from .goreleaser.yaml
rename to .goreleaser.yml
index 8bd4b40..d55eb1b 100644
--- a/.goreleaser.yaml
+++ b/.goreleaser.yml
@@ -7,20 +7,23 @@ before:
   hooks:
     - go mod tidy
     - go generate ./...
 
 builds:
   - env:
       - CGO_ENABLED=0
     goos:
       - linux
       - darwin
+    goarch:
+      - amd64
+      - arm64
 
 signs:
   - cmd: cosign
     stdin: "{{ .Env.COSIGN_PASSWORD }}"
     args:
       - "sign-blob"
       - "--key=cosign.key"
       - "--output-signature=${signature}"
       - "${artifact}"
       - "--yes" # needed on cosign 2.0.0+
diff --git a/cmd/db_migrations_test.go b/cmd/db_migrations_test.go
deleted file mode 100644
index ba16c00..0000000
--- a/cmd/db_migrations_test.go
+++ /dev/null
@@ -1,16 +0,0 @@
-package cmd
-
-import (
-	"testing"
-
-	"github.com/stretchr/testify/assert"
-)
-
-func TestMigrationsAreSetupCorrectly(t *testing.T) {
-	migrations := getMigrations()
-	for i := 2; i <= latestDBVersion; i++ {
-		m, ok := migrations[i]
-		assert.True(t, ok)
-		assert.NotEmpty(t, m)
-	}
-}
diff --git a/cmd/root.go b/cmd/root.go
index c1916bc..b3f47d4 100644
--- a/cmd/root.go
+++ b/cmd/root.go
@@ -1,310 +1,386 @@
 package cmd
 
 import (
 	"bufio"
 	"database/sql"
 	"errors"
 	"fmt"
 	"io/fs"
 	"math/rand"
 	"os"
-	"os/user"
+	"path/filepath"
 	"strings"
 
+	pers "github.com/dhth/hours/internal/persistence"
 	"github.com/dhth/hours/internal/ui"
 	"github.com/spf13/cobra"
 )
 
 const (
-	author        = "@dhth"
-	repoIssuesUrl = "https://github.com/dhth/hours/issues"
+	defaultDBName     = "hours.db"
+	author            = "@dhth"
+	repoIssuesURL     = "https://github.com/dhth/hours/issues"
+	numDaysThreshold  = 30
+	numTasksThreshold = 20
 )
 
 var (
-	dbPath              string
-	db                  *sql.DB
-	reportAgg           bool
-	recordsInteractive  bool
-	recordsOutputPlain  bool
-	activeTemplate      string
-	genNumDays          uint8
-	genNumTasks         uint8
-	genSkipConfirmation bool
+	errCouldntGetHomeDir        = errors.New("couldn't get home directory")
+	errDBFileExtIncorrect       = errors.New("db file needs to end with .db")
+	errCouldntCreateDBDirectory = errors.New("couldn't create directory for database")
+	errCouldntCreateDB          = errors.New("couldn't create database")
+	errCouldntInitializeDB      = errors.New("couldn't initialize database")
+	errCouldntOpenDB            = errors.New("couldn't open database")
+	errCouldntGenerateData      = errors.New("couldn't generate dummy data")
+	errNumDaysExceedsThreshold  = errors.New("number of days exceeds threshold")
+	errNumTasksExceedsThreshold = errors.New("number of tasks exceeds threshold")
+	errCouldntReadInput         = errors.New("couldn't read input")
+	errIncorrectCodeEntered     = errors.New("incorrect code entered")
+
+	msgReportIssue = fmt.Sprintf("This isn't supposed to happen; let %s know about this error via \n%s.", author, repoIssuesURL)
 )
 
-func die(msg string, args ...any) {
-	fmt.Fprintf(os.Stderr, msg+"\n", args...)
-	os.Exit(1)
-}
-
-func setupDB() {
-	if dbPath == "" {
-		die("dbpath cannot be empty")
+func Execute() error {
+	rootCmd, err := NewRootCommand()
+	if err != nil {
+		fmt.Fprintf(os.Stderr, "Error: %s\n", err)
+		if errors.Is(err, errCouldntGetHomeDir) {
+			fmt.Printf("\n%s\n", msgReportIssue)
+		}
+		return err
 	}
 
-	dbPathFull := expandTilde(dbPath)
+	err = rootCmd.Execute()
+	if errors.Is(err, errCouldntGenerateData) {
+		fmt.Printf("\n%s\n", msgReportIssue)
+	}
+	return err
+}
 
+func setupDB(dbPathFull string) (*sql.DB, error) {
+	var db *sql.DB
 	var err error
 
 	_, err = os.Stat(dbPathFull)
 	if errors.Is(err, fs.ErrNotExist) {
-		db, err = getDB(dbPathFull)
-		if err != nil {
-			die(`Couldn't create hours' local database. This is a fatal error;
-let %s know about this via %s.
 
-Error: %s`,
-				author,
-				repoIssuesUrl,
-				err)
+		dir := filepath.Dir(dbPathFull)
+		err = os.MkdirAll(dir, 0o755)
+		if err != nil {
+			return nil, fmt.Errorf("%w: %s", errCouldntCreateDBDirectory, err.Error())
 		}
 
-		err = initDB(db)
+		db, err = pers.GetDB(dbPathFull)
 		if err != nil {
-			die(`Couldn't create hours' local database. This is a fatal error;
-let %s know about this via %s.
-
-Error: %s`,
-				author,
-				repoIssuesUrl,
-				err)
+			return nil, fmt.Errorf("%w: %s", errCouldntCreateDB, err.Error())
+		}
+
+		err = pers.InitDB(db)
+		if err != nil {
+			return nil, fmt.Errorf("%w: %s", errCouldntInitializeDB, err.Error())
+		}
+		err = pers.UpgradeDB(db, 1)
+		if err != nil {
+			return nil, err
 		}
-		upgradeDB(db, 1)
 	} else {
-		db, err = getDB(dbPathFull)
+		db, err = pers.GetDB(dbPathFull)
 		if err != nil {
-			die(`Couldn't open hours' local database. This is a fatal error;
-let %s know about this via %s.
-
-Error: %s`,
-				author,
-				repoIssuesUrl,
-				err)
+			return nil, fmt.Errorf("%w: %s", errCouldntOpenDB, err.Error())
+		}
+		err = pers.UpgradeDBIfNeeded(db)
+		if err != nil {
+			return nil, err
 		}
-		upgradeDBIfNeeded(db)
 	}
+
+	return db, nil
 }
 
-var rootCmd = &cobra.Command{
-	Use:   "hours",
-	Short: "\"hours\" is a no-frills time tracking toolkit for the command line",
-	Long: `"hours" is a no-frills time tracking toolkit for the command line.
+func NewRootCommand() (*cobra.Command, error) {
+	var (
+		userHomeDir         string
+		dbPath              string
+		dbPathFull          string
+		db                  *sql.DB
+		reportAgg           bool
+		recordsInteractive  bool
+		recordsOutputPlain  bool
+		activeTemplate      string
+		genNumDays          uint8
+		genNumTasks         uint8
+		genSkipConfirmation bool
+	)
+
+	rootCmd := &cobra.Command{
+		Use:   "hours",
+		Short: "\"hours\" is a no-frills time tracking toolkit for the command line",
+		Long: `"hours" is a no-frills time tracking toolkit for the command line.
 
 You can use "hours" to track time on your tasks, or view logs, reports, and
 summary statistics for your tracked time.
 `,
-	PersistentPreRun: func(cmd *cobra.Command, args []string) {
-		if cmd.CalledAs() == "gen" {
-			return
-		}
-		setupDB()
-	},
-	Run: func(cmd *cobra.Command, args []string) {
-		ui.RenderUI(db)
-	},
-}
+		SilenceUsage: true,
+		PersistentPreRunE: func(cmd *cobra.Command, _ []string) error {
+			if cmd.CalledAs() == "updates" {
+				return nil
+			}
 
-var generateCmd = &cobra.Command{
-	Use:   "gen",
-	Short: "Generate dummy log entries (helpful for beginners)",
-	Long: `Generate dummy log entries.
+			dbPathFull = expandTilde(dbPath, userHomeDir)
+			if filepath.Ext(dbPathFull) != ".db" {
+				return errDBFileExtIncorrect
+			}
+
+			var err error
+			db, err = setupDB(dbPathFull)
+			switch {
+			case errors.Is(err, errCouldntCreateDB):
+				fmt.Fprintf(os.Stderr, `Couldn't create omm's local database.
+%s
+
+`, msgReportIssue)
+			case errors.Is(err, errCouldntInitializeDB):
+				fmt.Fprintf(os.Stderr, `Couldn't initialise omm's local database.
+%s
+
+`, msgReportIssue)
+				// cleanup
+				cleanupErr := os.Remove(dbPathFull)
+				if cleanupErr != nil {
+					fmt.Fprintf(os.Stderr, `Failed to remove omm's database file as well (at %s). Remove it manually.
+Clean up error: %s
+
+`, dbPathFull, cleanupErr.Error())
+				}
+			case errors.Is(err, errCouldntOpenDB):
+				fmt.Fprintf(os.Stderr, `Couldn't open omm's local database.
+%s
+
+`, msgReportIssue)
+			case errors.Is(err, pers.ErrCouldntFetchDBVersion):
+				fmt.Fprintf(os.Stderr, `Couldn't get omm's latest database version.
+%s
+
+`, msgReportIssue)
+			case errors.Is(err, pers.ErrDBDowngraded):
+				fmt.Fprintf(os.Stderr, `Looks like you downgraded omm. You should either delete omm's database file (you
+will lose data by doing that), or upgrade omm to the latest version.
+
+`)
+			case errors.Is(err, pers.ErrDBMigrationFailed):
+				fmt.Fprintf(os.Stderr, `Something went wrong migrating omm's database.
+
+You can try running omm by passing it a custom database file path (using
+--db-path; this will create a new database) to see if that fixes things. If that
+works, you can either delete the previous database, or keep using this new
+database (both are not ideal).
+
+%s
+Sorry for breaking the upgrade step!
+
+---
+
+`, msgReportIssue)
+			}
+
+			if err != nil {
+				return err
+			}
+
+			return nil
+		},
+		RunE: func(_ *cobra.Command, _ []string) error {
+			return ui.RenderUI(db)
+		},
+	}
+
+	generateCmd := &cobra.Command{
+		Use:   "gen",
+		Short: "Generate dummy log entries (helpful for beginners)",
+		Long: `Generate dummy log entries.
 This is intended for new users of 'hours' so they can get a sense of its
 capabilities without actually tracking any time. It's recommended to always use
 this with a --dbpath/-d flag that points to a throwaway database.
 `,
-	Run: func(cmd *cobra.Command, args []string) {
-		if genNumDays > 30 {
-			die("Maximum value for number of days is 30")
-		}
-		if genNumTasks > 20 {
-			die("Maximum value for number of days is 20")
-		}
+		RunE: func(_ *cobra.Command, _ []string) error {
+			if genNumDays > numDaysThreshold {
+				return fmt.Errorf("%w (%d)", errNumDaysExceedsThreshold, numDaysThreshold)
+			}
+			if genNumTasks > numTasksThreshold {
+				return fmt.Errorf("%w (%d)", errNumTasksExceedsThreshold, numTasksThreshold)
+			}
 
-		dbPathFull := expandTilde(dbPath)
-
-		_, statErr := os.Stat(dbPathFull)
-		if statErr == nil {
-			die(`A file already exists at %s. Either delete it, or use a different path.
-
-Tip: 'gen' should always be used on a throwaway database file.`, dbPathFull)
-		}
-
-		if !genSkipConfirmation {
-			fmt.Print(ui.WarningStyle.Render(`
+			if !genSkipConfirmation {
+				fmt.Print(ui.WarningStyle.Render(`
 WARNING: You shouldn't run 'gen' on hours' actively used database as it'll
-create dummy entries in it. You can run it out on a throwaway database by
-passing a path for it via --dbpath/-d (use it for all further invocations of
-'hours' as well).
+create dummy entries in it. You can run it on a throwaway database by passing a
+path for it via --dbpath/-d (use it for all further invocations of 'hours' as
+well).
 `))
-			fmt.Print(`
+				fmt.Print(`
 The 'gen' subcommand is intended for new users of 'hours' so they can get a
 sense of its capabilities without actually tracking any time.
 
 ---
 
 `)
-			confirm := getConfirmation()
-			if !confirm {
-				fmt.Printf("\nIncorrect code; exiting\n")
-				os.Exit(1)
+				confirm, err := getConfirmation()
+				if err != nil {
+					return err
+				}
+				if !confirm {
+					return fmt.Errorf("%w", errIncorrectCodeEntered)
+				}
 			}
-		}
 
-		setupDB()
-		genErr := ui.GenerateData(db, genNumDays, genNumTasks)
-		if genErr != nil {
-			die(`Something went wrong generating dummy data.
-let %s know about this via %s.
-
-Error: %s`, author, repoIssuesUrl, genErr)
-		}
-		fmt.Printf(`
+			genErr := ui.GenerateData(db, genNumDays, genNumTasks)
+			if genErr != nil {
+				return fmt.Errorf("%w: %s", errCouldntGenerateData, genErr.Error())
+			}
+			fmt.Printf(`
 Successfully generated dummy data in the database file: %s
 
 If this is not the default database file path, use --dbpath/-d with 'hours' when
 you want to access the dummy data.
 
 Go ahead and try the following!
 
 hours --dbpath=%s
 hours --dbpath=%s report week -i
 hours --dbpath=%s log today -i
 hours --dbpath=%s stats today -i
 `, dbPath, dbPath, dbPath, dbPath, dbPath)
-	},
-}
+			return nil
+		},
+	}
 
-var reportCmd = &cobra.Command{
-	Use:   "report",
-	Short: "Output a report based on task log entries",
-	Long: `Output a report based on task log entries.
+	reportCmd := &cobra.Command{
+		Use:   "report",
+		Short: "Output a report based on task log entries",
+		Long: `Output a report based on task log entries.
 
 Reports show time spent on tasks per day in the time period you specify. These
 can also be aggregated (using -a) to consolidate all task entries and show the
 cumulative time spent on each task per day.
 
 Accepts an argument, which can be one of the following:
 
   today:     for today's report
   yest:      for yesterday's report
   3d:        for a report on the last 3 days (default)
   week:      for a report on the current week
   date:      for a report for a specific date (eg. "2024/06/08")
   range:     for a report for a date range (eg. "2024/06/08...2024/06/12")
 
 Note: If a task log continues past midnight in your local timezone, it
 will be reported on the day it ends.
 `,
-	Args: cobra.MaximumNArgs(1),
-	Run: func(cmd *cobra.Command, args []string) {
-		var period string
-		if len(args) == 0 {
-			period = "3d"
-		} else {
-			period = args[0]
-		}
+		Args: cobra.MaximumNArgs(1),
+		RunE: func(_ *cobra.Command, args []string) error {
+			var period string
+			if len(args) == 0 {
+				period = "3d"
+			} else {
+				period = args[0]
+			}
 
-		ui.RenderReport(db, os.Stdout, recordsOutputPlain, period, reportAgg, recordsInteractive)
-	},
-}
+			return ui.RenderReport(db, os.Stdout, recordsOutputPlain, period, reportAgg, recordsInteractive)
+		},
+	}
 
-var logCmd = &cobra.Command{
-	Use:   "log",
-	Short: "Output task log entries",
-	Long: `Output task log entries.
+	logCmd := &cobra.Command{
+		Use:   "log",
+		Short: "Output task log entries",
+		Long: `Output task log entries.
 
 Accepts an argument, which can be one of the following:
 
   today:     for log entries from today (default)
   yest:      for log entries from yesterday
   3d:        for log entries from the last 3 days
   week:      for log entries from the current week
   date:      for log entries from a specific date (eg. "2024/06/08")
   range:     for log entries from a specific date range (eg. "2024/06/08...2024/06/12")
 
 Note: If a task log continues past midnight in your local timezone, it'll
 appear in the log for the day it ends.
 `,
-	Args: cobra.MaximumNArgs(1),
-	Run: func(cmd *cobra.Command, args []string) {
-		var period string
-		if len(args) == 0 {
-			period = "today"
-		} else {
-			period = args[0]
-		}
+		Args: cobra.MaximumNArgs(1),
+		RunE: func(_ *cobra.Command, args []string) error {
+			var period string
+			if len(args) == 0 {
+				period = "today"
+			} else {
+				period = args[0]
+			}
 
-		ui.RenderTaskLog(db, os.Stdout, recordsOutputPlain, period, recordsInteractive)
-	},
-}
+			return ui.RenderTaskLog(db, os.Stdout, recordsOutputPlain, period, recordsInteractive)
+		},
+	}
 
-var statsCmd = &cobra.Command{
-	Use:   "stats",
-	Short: "Output statistics for tracked time",
-	Long: `Output statistics for tracked time.
+	statsCmd := &cobra.Command{
+		Use:   "stats",
+		Short: "Output statistics for tracked time",
+		Long: `Output statistics for tracked time.
 
 Accepts an argument, which can be one of the following:
 
   today:     show stats for today
   yest:      show stats for yesterday
   3d:        show stats for the last 3 days (default)
   week:      show stats for the current week
   date:      show stats for a specific date (eg. "2024/06/08")
   range:     show stats for a specific date range (eg. "2024/06/08...2024/06/12")
   all:       show stats for all log entries
 
 Note: If a task log continues past midnight in your local timezone, it'll
 be considered in the stats for the day it ends.
 `,
-	Args: cobra.MaximumNArgs(1),
-	Run: func(cmd *cobra.Command, args []string) {
-		var period string
-		if len(args) == 0 {
-			period = "3d"
-		} else {
-			period = args[0]
-		}
+		Args: cobra.MaximumNArgs(1),
+		RunE: func(_ *cobra.Command, args []string) error {
+			var period string
+			if len(args) == 0 {
+				period = "3d"
+			} else {
+				period = args[0]
+			}
 
-		ui.RenderStats(db, os.Stdout, recordsOutputPlain, period, recordsInteractive)
-	},
-}
+			return ui.RenderStats(db, os.Stdout, recordsOutputPlain, period, recordsInteractive)
+		},
+	}
 
-var activeCmd = &cobra.Command{
-	Use:   "active",
-	Short: "Show the task being actively tracked by \"hours\"",
-	Long: `Show the task being actively tracked by "hours".
+	activeCmd := &cobra.Command{
+		Use:   "active",
+		Short: "Show the task being actively tracked by \"hours\"",
+		Long: `Show the task being actively tracked by "hours".
 
 You can pass in a template using the --template/-t flag, which supports the
 following placeholders:
 
   {{task}}:  for the task summary
   {{time}}:  for the time spent so far on the active log entry
 
 eg. hours active -t ' {{task}} ({{time}}) '
 `,
-	Run: func(cmd *cobra.Command, args []string) {
-		ui.ShowActiveTask(db, os.Stdout, activeTemplate)
-	},
-}
-
-func init() {
-	currentUser, err := user.Current()
-	if err != nil {
-		die(`Couldn't get your home directory. This is a fatal error;
-use --dbpath to specify database path manually
-let %s know about this via %s.
-
-Error: %s`, author, repoIssuesUrl, err)
+		RunE: func(_ *cobra.Command, _ []string) error {
+			return ui.ShowActiveTask(db, os.Stdout, activeTemplate)
+		},
 	}
 
-	defaultDBPath := fmt.Sprintf("%s/hours.db", currentUser.HomeDir)
+	var err error
+	userHomeDir, err = os.UserHomeDir()
+	if err != nil {
+		return nil, fmt.Errorf("%w: %s", errCouldntGetHomeDir, err.Error())
+	}
+
+	defaultDBPath := filepath.Join(userHomeDir, defaultDBName)
 	rootCmd.PersistentFlags().StringVarP(&dbPath, "dbpath", "d", defaultDBPath, "location of hours' database file")
 
 	generateCmd.Flags().Uint8Var(&genNumDays, "num-days", 30, "number of days to generate fake data for")
 	generateCmd.Flags().Uint8Var(&genNumTasks, "num-tasks", 10, "number of tasks to generate fake data for")
 	generateCmd.Flags().BoolVarP(&genSkipConfirmation, "yes", "y", false, "to skip confirmation")
 
 	reportCmd.Flags().BoolVarP(&reportAgg, "agg", "a", false, "whether to aggregate data by task for each day in report")
 	reportCmd.Flags().BoolVarP(&recordsInteractive, "interactive", "i", false, "whether to view report interactively")
 	reportCmd.Flags().BoolVarP(&recordsOutputPlain, "plain", "p", false, "whether to output report without any formatting")
 
@@ -316,43 +392,38 @@ Error: %s`, author, repoIssuesUrl, err)
 
 	activeCmd.Flags().StringVarP(&activeTemplate, "template", "t", ui.ActiveTaskPlaceholder, "string template to use for outputting active task")
 
 	rootCmd.AddCommand(generateCmd)
 	rootCmd.AddCommand(reportCmd)
 	rootCmd.AddCommand(logCmd)
 	rootCmd.AddCommand(statsCmd)
 	rootCmd.AddCommand(activeCmd)
 
 	rootCmd.CompletionOptions.DisableDefaultCmd = true
-}
 
-func Execute() {
-	err := rootCmd.Execute()
-	if err != nil {
-		die("Something went wrong: %s", err)
-	}
+	return rootCmd, nil
 }
 
 func getRandomChars(length int) string {
 	const charset = "abcdefghijklmnopqrstuvwxyz"
 
 	var code string
 	for i := 0; i < length; i++ {
 		code += string(charset[rand.Intn(len(charset))])
 	}
 	return code
 }
 
-func getConfirmation() bool {
+func getConfirmation() (bool, error) {
 	code := getRandomChars(2)
 	reader := bufio.NewReader(os.Stdin)
 
 	fmt.Printf("Type %s to proceed: ", code)
 
 	response, err := reader.ReadString('\n')
 	if err != nil {
-		die("Something went wrong reading input: %s", err)
+		return false, fmt.Errorf("%w: %s", errCouldntReadInput, err.Error())
 	}
 	response = strings.TrimSpace(response)
 
-	return response == code
+	return response == code, nil
 }
diff --git a/cmd/utils.go b/cmd/utils.go
index ea1318e..62349f7 100644
--- a/cmd/utils.go
+++ b/cmd/utils.go
@@ -1,18 +1,14 @@
 package cmd
 
 import (
-	"os"
-	"os/user"
+	"path/filepath"
 	"strings"
 )
 
-func expandTilde(path string) string {
-	if strings.HasPrefix(path, "~") {
-		usr, err := user.Current()
-		if err != nil {
-			os.Exit(1)
-		}
-		return strings.Replace(path, "~", usr.HomeDir, 1)
+func expandTilde(path string, homeDir string) string {
+	pathWithoutTilde, found := strings.CutPrefix(path, "~/")
+	if !found {
+		return path
 	}
-	return path
+	return filepath.Join(homeDir, pathWithoutTilde)
 }
diff --git a/cmd/utils_test.go b/cmd/utils_test.go
new file mode 100644
index 0000000..9d73a0d
--- /dev/null
+++ b/cmd/utils_test.go
@@ -0,0 +1,37 @@
+package cmd
+
+import (
+	"testing"
+
+	"github.com/stretchr/testify/assert"
+)
+
+func TestExpandTilde(t *testing.T) {
+	testCases := []struct {
+		name     string
+		path     string
+		homeDir  string
+		expected string
+	}{
+		{
+			name:     "a simple case",
+			path:     "~/some/path",
+			homeDir:  "/Users/trinity",
+			expected: "/Users/trinity/some/path",
+		},
+		{
+			name:     "path with no ~",
+			path:     "some/path",
+			homeDir:  "/Users/trinity",
+			expected: "some/path",
+		},
+	}
+
+	for _, tt := range testCases {
+		t.Run(tt.name, func(t *testing.T) {
+			got := expandTilde(tt.path, tt.homeDir)
+
+			assert.Equal(t, tt.expected, got)
+		})
+	}
+}
diff --git a/cmd/db.go b/internal/persistence/init.go
similarity index 87%
rename from cmd/db.go
rename to internal/persistence/init.go
index bb572d5..3d71182 100644
--- a/cmd/db.go
+++ b/internal/persistence/init.go
@@ -1,25 +1,18 @@
-package cmd
+package persistence
 
 import (
 	"database/sql"
 	"time"
 )
 
-func getDB(dbpath string) (*sql.DB, error) {
-	db, err := sql.Open("sqlite", dbpath)
-	db.SetMaxOpenConns(1)
-	db.SetMaxIdleConns(1)
-	return db, err
-}
-
-func initDB(db *sql.DB) error {
+func InitDB(db *sql.DB) error {
 	// these init queries cannot be changed
 	// once hours is released; only further migrations
 	// can be added, which are run whenever hours
 	// sees a difference between the values in db_versions
 	// and latestDBVersion
 	_, err := db.Exec(`
 CREATE TABLE IF NOT EXISTS db_versions (
     id INTEGER PRIMARY KEY AUTOINCREMENT,
     version INTEGER NOT NULL,
     created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
diff --git a/cmd/db_migrations.go b/internal/persistence/migrations.go
similarity index 57%
rename from cmd/db_migrations.go
rename to internal/persistence/migrations.go
index e36e1b7..d10f004 100644
--- a/cmd/db_migrations.go
+++ b/internal/persistence/migrations.go
@@ -1,35 +1,41 @@
-package cmd
+package persistence
 
 import (
 	"database/sql"
+	"errors"
+	"fmt"
 	"time"
 )
 
-const (
-	latestDBVersion = 1 // only upgrade this after adding a migration in getMigrations
+const latestDBVersion = 1 // only upgrade this after adding a migration in getMigrations
+
+var (
+	ErrDBDowngraded          = errors.New("database downgraded")
+	ErrDBMigrationFailed     = errors.New("database migration failed")
+	ErrCouldntFetchDBVersion = errors.New("couldn't fetch version")
 )
 
 type dbVersionInfo struct {
 	id        int
 	version   int
 	createdAt time.Time
 }
 
 func getMigrations() map[int]string {
 	migrations := make(map[int]string)
 	// these migrations should not be modified once released.
 	// that is, migrations is an append-only map.
 
 	// migrations[2] = `
 	// ALTER TABLE task
-	//     ADD COLUMN a_col INTEGER NOT NULL DEFAULT 1;
+	// ADD COLUMN new_col TEXT;
 	// `
 
 	return migrations
 }
 
 func fetchLatestDBVersion(db *sql.DB) (dbVersionInfo, error) {
 	row := db.QueryRow(`
 SELECT id, version, created_at
 FROM db_versions
 ORDER BY created_at DESC
@@ -39,65 +45,54 @@ LIMIT 1;
 	var dbVersion dbVersionInfo
 	err := row.Scan(
 		&dbVersion.id,
 		&dbVersion.version,
 		&dbVersion.createdAt,
 	)
 
 	return dbVersion, err
 }
 
-func upgradeDBIfNeeded(db *sql.DB) {
-	latestVersionInDB, versionErr := fetchLatestDBVersion(db)
-	if versionErr != nil {
-		die(`Couldn't get hours' latest database version. This is a fatal error; let %s
-know about this via %s.
-
-Error: %s`,
-			author,
-			repoIssuesUrl,
-			versionErr)
+func UpgradeDBIfNeeded(db *sql.DB) error {
+	latestVersionInDB, err := fetchLatestDBVersion(db)
+	if err != nil {
+		return fmt.Errorf("%w: %s", ErrCouldntFetchDBVersion, err.Error())
 	}
 
 	if latestVersionInDB.version > latestDBVersion {
-		die(`Looks like you downgraded hours. You should either delete hours'
-database file (you will lose data by doing that), or upgrade hours to
-the latest version.`)
+		return fmt.Errorf("%w; debug info: version=%d, created at=%q)",
+			ErrDBDowngraded,
+			latestVersionInDB.version,
+			latestVersionInDB.createdAt.Format(time.RFC3339),
+		)
 	}
 
 	if latestVersionInDB.version < latestDBVersion {
-		upgradeDB(db, latestVersionInDB.version)
+		err = UpgradeDB(db, latestVersionInDB.version)
+		if err != nil {
+			return err
+		}
 	}
+
+	return nil
 }
 
-func upgradeDB(db *sql.DB, currentVersion int) {
+func UpgradeDB(db *sql.DB, currentVersion int) error {
 	migrations := getMigrations()
 	for i := currentVersion + 1; i <= latestDBVersion; i++ {
 		migrateQuery := migrations[i]
 		migrateErr := runMigration(db, migrateQuery, i)
 		if migrateErr != nil {
-			die(`Something went wrong migrating hours' database to version %d. This is not
-supposed to happen. You can try running hours by passing it a custom database
-file path (using --dbpath; this will create a new database) to see if that fixes
-things. If that works, you can either delete the previous database, or keep
-using this new database (both are not ideal).
-
-If you can, let %s know about this error via
-%s.
-Sorry for breaking the upgrade step!
-
----
-
-Error: %s
-`, i, author, repoIssuesUrl, migrateErr)
+			return fmt.Errorf("%w (version %d): %v", ErrDBMigrationFailed, i, migrateErr.Error())
 		}
 	}
+	return nil
 }
 
 func runMigration(db *sql.DB, migrateQuery string, version int) error {
 	tx, err := db.Begin()
 	if err != nil {
 		return err
 	}
 	defer func() {
 		_ = tx.Rollback()
 	}()
diff --git a/internal/persistence/migrations_test.go b/internal/persistence/migrations_test.go
new file mode 100644
index 0000000..f97ac67
--- /dev/null
+++ b/internal/persistence/migrations_test.go
@@ -0,0 +1,69 @@
+package persistence
+
+import (
+	"database/sql"
+	"testing"
+
+	"github.com/stretchr/testify/assert"
+	_ "modernc.org/sqlite" // sqlite driver
+)
+
+func TestMigrationsAreSetupCorrectly(t *testing.T) {
+	// GIVEN
+	// WHEN
+	migrations := getMigrations()
+
+	// THEN
+	for i := 2; i <= latestDBVersion; i++ {
+		m, ok := migrations[i]
+		if !ok {
+			assert.True(t, ok, "couldn't get migration %d", i)
+		}
+		if m == "" {
+			assert.NotEmpty(t, ok, "migration %d is empty", i)
+		}
+	}
+}
+
+func TestMigrationsWork(t *testing.T) {
+	// GIVEN
+	var testDB *sql.DB
+	var err error
+	testDB, err = sql.Open("sqlite", ":memory:")
+	if err != nil {
+		t.Fatalf("Couldn't open database: %s", err.Error())
+	}
+
+	err = InitDB(testDB)
+	if err != nil {
+		t.Fatalf("Couldn't initialize database: %s", err.Error())
+	}
+
+	// WHEN
+	err = UpgradeDB(testDB, 1)
+
+	// THEN
+	assert.NoError(t, err)
+}
+
+func TestRunMigrationFailsWhenGivenBadMigration(t *testing.T) {
+	// GIVEN
+	var testDB *sql.DB
+	var err error
+	testDB, err = sql.Open("sqlite", ":memory:")
+	if err != nil {
+		t.Fatalf("Couldn't open database: %s", err.Error())
+	}
+
+	err = InitDB(testDB)
+	if err != nil {
+		t.Fatalf("Couldn't initialize database: %s", err.Error())
+	}
+
+	// WHEN
+	query := "BAD SQL CODE;"
+	migrateErr := runMigration(testDB, query, 1)
+
+	// THEN
+	assert.Error(t, migrateErr)
+}
diff --git a/internal/persistence/open.go b/internal/persistence/open.go
new file mode 100644
index 0000000..6251f9f
--- /dev/null
+++ b/internal/persistence/open.go
@@ -0,0 +1,12 @@
+package persistence
+
+import (
+	"database/sql"
+)
+
+func GetDB(dbpath string) (*sql.DB, error) {
+	db, err := sql.Open("sqlite", dbpath)
+	db.SetMaxOpenConns(1)
+	db.SetMaxIdleConns(1)
+	return db, err
+}
diff --git a/internal/persistence/queries.go b/internal/persistence/queries.go
new file mode 100644
index 0000000..73f6b05
--- /dev/null
+++ b/internal/persistence/queries.go
@@ -0,0 +1,615 @@
+package persistence
+
+import (
+	"database/sql"
+	"errors"
+	"fmt"
+	"time"
+
+	"github.com/dhth/hours/internal/types"
+)
+
+var ErrCouldntRollBackTx = errors.New("couldn't roll back transaction")
+
+func InsertNewTL(db *sql.DB, taskID int, beginTs time.Time) (int, error) {
+	return runInTxAndReturnID(db, func(tx *sql.Tx) (int, error) {
+		stmt, err := tx.Prepare(`
+INSERT INTO task_log (task_id, begin_ts, active)
+VALUES (?, ?, ?);
+`)
+		if err != nil {
+			return -1, err
+		}
+		defer stmt.Close()
+
+		res, err := stmt.Exec(taskID, beginTs.UTC(), true)
+		if err != nil {
+			return -1, err
+		}
+
+		lastID, err := res.LastInsertId()
+		if err != nil {
+			return -1, err
+		}
+
+		return int(lastID), nil
+	})
+}
+
+func UpdateTLBeginTS(db *sql.DB, beginTs time.Time) error {
+	stmt, err := db.Prepare(`
+UPDATE task_log SET begin_ts=?
+WHERE active is true;
+`)
+	if err != nil {
+		return err
+	}
+	defer stmt.Close()
+
+	_, err = stmt.Exec(beginTs.UTC(), true)
+	if err != nil {
+		return err
+	}
+
+	return nil
+}
+
+func DeleteActiveTL(db *sql.DB) error {
+	stmt, err := db.Prepare(`
+DELETE FROM task_log
+WHERE active=true;
+`)
+	if err != nil {
+		return err
+	}
+	defer stmt.Close()
+
+	_, err = stmt.Exec()
+
+	return err
+}
+
+func UpdateActiveTL(db *sql.DB, taskLogID int, taskID int, beginTs, endTs time.Time, secsSpent int, comment string) error {
+	return runInTx(db, func(tx *sql.Tx) error {
+		stmt, err := tx.Prepare(`
+UPDATE task_log
+SET active = 0,
+    begin_ts = ?,
+    end_ts = ?,
+    secs_spent = ?,
+    comment = ?
+WHERE id = ?
+AND active = 1;
+`)
+		if err != nil {
+			return err
+		}
+		defer stmt.Close()
+
+		_, err = stmt.Exec(beginTs.UTC(), endTs.UTC(), secsSpent, comment, taskLogID)
+		if err != nil {
+			return err
+		}
+
+		tStmt, err := tx.Prepare(`
+UPDATE task
+SET secs_spent = secs_spent+?,
+    updated_at = ?
+WHERE id = ?;
+    `)
+		if err != nil {
+			return err
+		}
+		defer tStmt.Close()
+
+		_, err = tStmt.Exec(secsSpent, time.Now().UTC(), taskID)
+
+		return err
+	})
+}
+
+func InsertManualTL(db *sql.DB, taskID int, beginTs time.Time, endTs time.Time, comment string) (int, error) {
+	return runInTxAndReturnID(db, func(tx *sql.Tx) (int, error) {
+		stmt, err := tx.Prepare(`
+INSERT INTO task_log (task_id, begin_ts, end_ts, secs_spent, comment, active)
+VALUES (?, ?, ?, ?, ?, ?);
+`)
+		if err != nil {
+			return -1, err
+		}
+		defer stmt.Close()
+
+		secsSpent := int(endTs.Sub(beginTs).Seconds())
+
+		res, err := stmt.Exec(taskID, beginTs.UTC(), endTs.UTC(), secsSpent, comment, false)
+		if err != nil {
+			return -1, err
+		}
+
+		lastID, err := res.LastInsertId()
+		if err != nil {
+			return -1, err
+		}
+
+		tStmt, err := tx.Prepare(`
+UPDATE task
+SET secs_spent = secs_spent+?,
+    updated_at = ?
+WHERE id = ?;
+    `)
+		if err != nil {
+			return -1, err
+		}
+		defer tStmt.Close()
+
+		_, err = tStmt.Exec(secsSpent, time.Now().UTC(), taskID)
+		if err != nil {
+			return -1, err
+		}
+
+		return int(lastID), nil
+	})
+}
+
+func FetchActiveTask(db *sql.DB) (types.ActiveTaskDetails, error) {
+	row := db.QueryRow(`
+SELECT t.id, t.summary, tl.begin_ts
+FROM task_log tl left join task t on tl.task_id = t.id
+WHERE tl.active=true;
+`)
+
+	var activeTaskDetails types.ActiveTaskDetails
+	err := row.Scan(
+		&activeTaskDetails.TaskID,
+		&activeTaskDetails.TaskSummary,
+		&activeTaskDetails.LastLogEntryBeginTS,
+	)
+	if errors.Is(err, sql.ErrNoRows) {
+		activeTaskDetails.TaskID = -1
+		return activeTaskDetails, nil
+	} else if err != nil {
+		return activeTaskDetails, err
+	}
+	activeTaskDetails.LastLogEntryBeginTS = activeTaskDetails.LastLogEntryBeginTS.Local()
+	return activeTaskDetails, nil
+}
+
+func InsertTask(db *sql.DB, summary string) (int, error) {
+	return runInTxAndReturnID(db, func(tx *sql.Tx) (int, error) {
+		stmt, err := tx.Prepare(`
+INSERT into task (summary, active, created_at, updated_at)
+VALUES (?, true, ?, ?);
+`)
+		if err != nil {
+			return -1, err
+		}
+		defer stmt.Close()
+
+		now := time.Now().UTC()
+		res, err := stmt.Exec(summary, now, now)
+		if err != nil {
+			return -1, err
+		}
+
+		lastID, err := res.LastInsertId()
+		if err != nil {
+			return -1, err
+		}
+
+		return int(lastID), nil
+	})
+}
+
+func UpdateTask(db *sql.DB, id int, summary string) error {
+	stmt, err := db.Prepare(`
+UPDATE task
+SET summary = ?,
+    updated_at = ?
+WHERE id = ?
+`)
+	if err != nil {
+		return err
+	}
+	defer stmt.Close()
+
+	_, err = stmt.Exec(summary, time.Now().UTC(), id)
+	if err != nil {
+		return err
+	}
+	return nil
+}
+
+func UpdateTaskActiveStatus(db *sql.DB, id int, active bool) error {
+	stmt, err := db.Prepare(`
+UPDATE task
+SET active = ?,
+    updated_at = ?
+WHERE id = ?
+`)
+	if err != nil {
+		return err
+	}
+	defer stmt.Close()
+
+	_, err = stmt.Exec(active, time.Now().UTC(), id)
+	if err != nil {
+		return err
+	}
+	return nil
+}
+
+func UpdateTaskData(db *sql.DB, t *types.Task) error {
+	row := db.QueryRow(`
+SELECT secs_spent, updated_at
+FROM task
+WHERE id=?;
+    `, t.ID)
+
+	err := row.Scan(
+		&t.SecsSpent,
+		&t.UpdatedAt,
+	)
+	if err != nil {
+		return err
+	}
+	return nil
+}
+
+func FetchTasks(db *sql.DB, active bool, limit int) ([]types.Task, error) {
+	var tasks []types.Task
+
+	rows, err := db.Query(`
+SELECT id, summary, secs_spent, created_at, updated_at, active
+FROM task
+WHERE active=?
+ORDER by updated_at DESC
+LIMIT ?;
+    `, active, limit)
+	if err != nil {
+		return nil, err
+	}
+	defer rows.Close()
+
+	for rows.Next() {
+		var entry types.Task
+		err = rows.Scan(&entry.ID,
+			&entry.Summary,
+			&entry.SecsSpent,
+			&entry.CreatedAt,
+			&entry.UpdatedAt,
+			&entry.Active,
+		)
+		if err != nil {
+			return nil, err
+		}
+		entry.CreatedAt = entry.CreatedAt.Local()
+		entry.UpdatedAt = entry.UpdatedAt.Local()
+		tasks = append(tasks, entry)
+
+	}
+	if rows.Err() != nil {
+		return nil, err
+	}
+	return tasks, nil
+}
+
+func FetchTLEntries(db *sql.DB, desc bool, limit int) ([]types.TaskLogEntry, error) {
+	var logEntries []types.TaskLogEntry
+
+	var order string
+	if desc {
+		order = "DESC"
+	} else {
+		order = "ASC"
+	}
+	query := fmt.Sprintf(`
+SELECT tl.id, tl.task_id, t.summary, tl.begin_ts, tl.end_ts, tl.secs_spent, tl.comment
+FROM task_log tl left join task t on tl.task_id=t.id
+WHERE tl.active=false
+ORDER by tl.begin_ts %s
+LIMIT ?;
+`, order)
+
+	rows, err := db.Query(query, limit)
+	if err != nil {
+		return nil, err
+	}
+	defer rows.Close()
+
+	for rows.Next() {
+		var entry types.TaskLogEntry
+		err = rows.Scan(&entry.ID,
+			&entry.TaskID,
+			&entry.TaskSummary,
+			&entry.BeginTS,
+			&entry.EndTS,
+			&entry.SecsSpent,
+			&entry.Comment,
+		)
+		if err != nil {
+			return nil, err
+		}
+		entry.BeginTS = entry.BeginTS.Local()
+		entry.EndTS = entry.EndTS.Local()
+		logEntries = append(logEntries, entry)
+
+	}
+	if rows.Err() != nil {
+		return nil, err
+	}
+	return logEntries, nil
+}
+
+func FetchTLEntriesBetweenTS(db *sql.DB, beginTs, endTs time.Time, limit int) ([]types.TaskLogEntry, error) {
+	var logEntries []types.TaskLogEntry
+
+	rows, err := db.Query(`
+SELECT tl.id, tl.task_id, t.summary, tl.begin_ts, tl.end_ts, tl.secs_spent, tl.comment
+FROM task_log tl left join task t on tl.task_id=t.id
+WHERE tl.active=false
+AND tl.end_ts >= ?
+AND tl.end_ts < ?
+ORDER by tl.begin_ts ASC LIMIT ?;
+    `, beginTs.UTC(), endTs.UTC(), limit)
+	if err != nil {
+		return nil, err
+	}
+	defer rows.Close()
+
+	for rows.Next() {
+		var entry types.TaskLogEntry
+		err = rows.Scan(&entry.ID,
+			&entry.TaskID,
+			&entry.TaskSummary,
+			&entry.BeginTS,
+			&entry.EndTS,
+			&entry.SecsSpent,
+			&entry.Comment,
+		)
+		if err != nil {
+			return nil, err
+		}
+		entry.BeginTS = entry.BeginTS.Local()
+		entry.EndTS = entry.EndTS.Local()
+		logEntries = append(logEntries, entry)
+
+	}
+	if rows.Err() != nil {
+		return nil, err
+	}
+	return logEntries, nil
+}
+
+func FetchStats(db *sql.DB, limit int) ([]types.TaskReportEntry, error) {
+	rows, err := db.Query(`
+SELECT tl.task_id, t.summary, COUNT(tl.id) as num_entries, t.secs_spent
+from task_log tl
+LEFT JOIN task t on tl.task_id = t.id
+GROUP BY tl.task_id
+ORDER BY t.secs_spent DESC
+limit ?;
+`, limit)
+	if err != nil {
+		return nil, err
+	}
+	defer rows.Close()
+
+	var tLE []types.TaskReportEntry
+
+	for rows.Next() {
+		var entry types.TaskReportEntry
+		err = rows.Scan(
+			&entry.TaskID,
+			&entry.TaskSummary,
+			&entry.NumEntries,
+			&entry.SecsSpent,
+		)
+		if err != nil {
+			return nil, err
+		}
+		tLE = append(tLE, entry)
+
+	}
+	if rows.Err() != nil {
+		return nil, err
+	}
+	return tLE, nil
+}
+
+func FetchStatsBetweenTS(db *sql.DB, beginTs, endTs time.Time, limit int) ([]types.TaskReportEntry, error) {
+	rows, err := db.Query(`
+SELECT tl.task_id, t.summary, COUNT(tl.id) as num_entries,  SUM(tl.secs_spent) AS secs_spent
+FROM task_log tl 
+LEFT JOIN task t ON tl.task_id = t.id
+WHERE tl.end_ts >= ? AND tl.end_ts < ?
+GROUP BY tl.task_id
+ORDER BY secs_spent DESC
+LIMIT ?;
+`, beginTs.UTC(), endTs.UTC(), limit)
+	if err != nil {
+		return nil, err
+	}
+	defer rows.Close()
+
+	var tLE []types.TaskReportEntry
+
+	for rows.Next() {
+		var entry types.TaskReportEntry
+		err = rows.Scan(
+			&entry.TaskID,
+			&entry.TaskSummary,
+			&entry.NumEntries,
+			&entry.SecsSpent,
+		)
+		if err != nil {
+			return nil, err
+		}
+		tLE = append(tLE, entry)
+
+	}
+	if rows.Err() != nil {
+		return nil, err
+	}
+	return tLE, nil
+}
+
+func FetchReportBetweenTS(db *sql.DB, beginTs, endTs time.Time, limit int) ([]types.TaskReportEntry, error) {
+	rows, err := db.Query(`
+SELECT tl.task_id, t.summary, COUNT(tl.id) as num_entries,  SUM(tl.secs_spent) AS secs_spent
+FROM task_log tl 
+LEFT JOIN task t ON tl.task_id = t.id
+WHERE tl.end_ts >= ? AND tl.end_ts < ?
+GROUP BY tl.task_id
+ORDER BY t.updated_at ASC
+LIMIT ?;
+`, beginTs.UTC(), endTs.UTC(), limit)
+	if err != nil {
+		return nil, err
+	}
+	defer rows.Close()
+
+	var tLE []types.TaskReportEntry
+
+	for rows.Next() {
+		var entry types.TaskReportEntry
+		err = rows.Scan(
+			&entry.TaskID,
+			&entry.TaskSummary,
+			&entry.NumEntries,
+			&entry.SecsSpent,
+		)
+		if err != nil {
+			return nil, err
+		}
+		tLE = append(tLE, entry)
+
+	}
+	if rows.Err() != nil {
+		return nil, err
+	}
+	return tLE, nil
+}
+
+func DeleteTaskLogEntry(db *sql.DB, entry *types.TaskLogEntry) error {
+	return runInTx(db, func(tx *sql.Tx) error {
+		stmt, err := tx.Prepare(`
+DELETE from task_log
+WHERE ID=?;
+`)
+		if err != nil {
+			return err
+		}
+		defer stmt.Close()
+
+		_, err = stmt.Exec(entry.ID)
+		if err != nil {
+			return err
+		}
+
+		tStmt, err := tx.Prepare(`
+UPDATE task
+SET secs_spent = secs_spent-?,
+    updated_at = ?
+WHERE id = ?;
+    `)
+		if err != nil {
+			return err
+		}
+		defer tStmt.Close()
+
+		_, err = tStmt.Exec(entry.SecsSpent, time.Now().UTC(), entry.TaskID)
+		return err
+	})
+}
+
+func runInTxAndReturnID(db *sql.DB, fn func(tx *sql.Tx) (int, error)) (int, error) {
+	tx, err := db.Begin()
+	if err != nil {
+		return -1, err
+	}
+
+	lastID, err := fn(tx)
+	if err == nil {
+		return lastID, tx.Commit()
+	}
+
+	rollbackErr := tx.Rollback()
+	if rollbackErr != nil {
+		return lastID, fmt.Errorf("%w: %w: %s", ErrCouldntRollBackTx, rollbackErr, err.Error())
+	}
+
+	return lastID, err
+}
+
+func runInTx(db *sql.DB, fn func(tx *sql.Tx) error) error {
+	tx, err := db.Begin()
+	if err != nil {
+		return err
+	}
+
+	err = fn(tx)
+	if err == nil {
+		return tx.Commit()
+	}
+
+	rollbackErr := tx.Rollback()
+	if rollbackErr != nil {
+		return fmt.Errorf("%w: %w: %w", ErrCouldntRollBackTx, rollbackErr, err)
+	}
+
+	return err
+}
+
+func fetchTaskByID(db *sql.DB, id int) (types.Task, error) {
+	var task types.Task
+	row := db.QueryRow(`
+SELECT id, summary, secs_spent, active, created_at, updated_at
+FROM task
+WHERE id=?;
+    `, id)
+
+	if row.Err() != nil {
+		return task, row.Err()
+	}
+	err := row.Scan(&task.ID,
+		&task.Summary,
+		&task.SecsSpent,
+		&task.Active,
+		&task.CreatedAt,
+		&task.UpdatedAt,
+	)
+	if err != nil {
+		return task, err
+	}
+	task.CreatedAt = task.CreatedAt.Local()
+	task.UpdatedAt = task.UpdatedAt.Local()
+
+	return task, nil
+}
+
+func fetchTaskLogByID(db *sql.DB, id int) (types.TaskLogEntry, error) {
+	var tl types.TaskLogEntry
+	row := db.QueryRow(`
+SELECT id, task_id, begin_ts, end_ts, secs_spent, comment
+FROM task_log
+WHERE id=?;
+    `, id)
+
+	if row.Err() != nil {
+		return tl, row.Err()
+	}
+	err := row.Scan(&tl.ID,
+		&tl.TaskID,
+		&tl.BeginTS,
+		&tl.EndTS,
+		&tl.SecsSpent,
+		&tl.Comment,
+	)
+	if err != nil {
+		return tl, err
+	}
+	tl.BeginTS = tl.BeginTS.Local()
+	tl.EndTS = tl.EndTS.Local()
+
+	return tl, nil
+}
diff --git a/internal/persistence/queries_test.go b/internal/persistence/queries_test.go
new file mode 100644
index 0000000..88fa033
--- /dev/null
+++ b/internal/persistence/queries_test.go
@@ -0,0 +1,361 @@
+package persistence
+
+import (
+	"database/sql"
+	"fmt"
+	"testing"
+	"time"
+
+	"github.com/dhth/hours/internal/types"
+	"github.com/stretchr/testify/assert"
+	"github.com/stretchr/testify/require"
+	_ "modernc.org/sqlite" // sqlite driver
+)
+
+const (
+	secsInOneHour  = 60 * 60
+	taskLogComment = "a task log outside the time range"
+)
+
+func TestRepository(t *testing.T) {
+	testDB, err := sql.Open("sqlite", ":memory:")
+	require.NoErrorf(t, err, "error opening DB: %v", err)
+
+	err = InitDB(testDB)
+	require.NoErrorf(t, err, "error initializing DB: %v", err)
+
+	err = UpgradeDB(testDB, 1)
+	require.NoErrorf(t, err, "error upgrading DB: %v", err)
+
+	t.Run("TestInsertTask", func(t *testing.T) {
+		t.Cleanup(func() { cleanupDB(t, testDB) })
+
+		// GIVEN
+		referenceTS := time.Now()
+		seedData := getTestData(referenceTS)
+		seedDB(t, testDB, seedData)
+
+		// WHEN
+		summary := "task 1"
+		taskID, err := InsertTask(testDB, summary)
+
+		// THEN
+		require.NoError(t, err, "failed to insert task")
+
+		task, fetchErr := fetchTaskByID(testDB, taskID)
+		require.NoError(t, fetchErr, "failed to fetch task")
+
+		assert.Equal(t, 3, task.ID)
+		assert.Equal(t, summary, task.Summary)
+		assert.True(t, task.Active)
+		assert.Zero(t, task.SecsSpent)
+	})
+
+	t.Run("TestUpdateActiveTL", func(t *testing.T) {
+		t.Cleanup(func() { cleanupDB(t, testDB) })
+
+		// GIVEN
+		referenceTS := time.Now()
+		seedData := getTestData(referenceTS)
+		seedDB(t, testDB, seedData)
+		taskID := 1
+		numSeconds := 60 * 90
+		endTS := time.Now()
+		beginTS := endTS.Add(time.Second * -1 * time.Duration(numSeconds))
+		tlID, insertErr := InsertNewTL(testDB, taskID, beginTS)
+		require.NoError(t, insertErr, "failed to insert task log")
+
+		taskBefore, err := fetchTaskByID(testDB, taskID)
+		require.NoError(t, err, "failed to fetch task")
+		numSecondsBefore := taskBefore.SecsSpent
+
+		// WHEN
+		comment := "a task log"
+		err = UpdateActiveTL(testDB, tlID, taskID, beginTS, endTS, numSeconds, comment)
+
+		// THEN
+		require.NoError(t, err, "failed to update task log")
+
+		taskLog, err := fetchTaskLogByID(testDB, tlID)
+		require.NoError(t, err, "failed to fetch task log")
+
+		taskAfter, err := fetchTaskByID(testDB, taskID)
+		require.NoError(t, err, "failed to fetch task")
+
+		assert.Equal(t, numSeconds, taskLog.SecsSpent)
+		assert.Equal(t, comment, taskLog.Comment)
+		assert.Equal(t, numSecondsBefore+numSeconds, taskAfter.SecsSpent)
+	})
+
+	t.Run("TestInsertManualTL", func(t *testing.T) {
+		t.Cleanup(func() { cleanupDB(t, testDB) })
+
+		// GIVEN
+		referenceTS := time.Now()
+		seedData := getTestData(referenceTS)
+		seedDB(t, testDB, seedData)
+		taskID := 1
+
+		taskBefore, err := fetchTaskByID(testDB, taskID)
+		require.NoError(t, err, "failed to fetch task")
+		numSecondsBefore := taskBefore.SecsSpent
+
+		// WHEN
+		comment := "a task log"
+		numSeconds := 60 * 90
+		endTS := time.Now()
+		beginTS := endTS.Add(time.Second * -1 * time.Duration(numSeconds))
+		tlID, err := InsertManualTL(testDB, taskID, beginTS, endTS, comment)
+
+		// THEN
+		require.NoError(t, err, "failed to insert task log")
+
+		taskLog, err := fetchTaskLogByID(testDB, tlID)
+		require.NoError(t, err, "failed to fetch task log")
+
+		taskAfter, err := fetchTaskByID(testDB, taskID)
+		require.NoError(t, err, "failed to fetch task")
+
+		assert.Equal(t, numSeconds, taskLog.SecsSpent)
+		assert.Equal(t, comment, taskLog.Comment)
+		assert.Equal(t, numSecondsBefore+numSeconds, taskAfter.SecsSpent)
+	})
+
+	t.Run("TestDeleteTaskLogEntry", func(t *testing.T) {
+		t.Cleanup(func() { cleanupDB(t, testDB) })
+
+		// GIVEN
+		referenceTS := time.Now()
+		seedData := getTestData(referenceTS)
+		seedDB(t, testDB, seedData)
+		taskID := 1
+		tlID := 1
+		taskBefore, err := fetchTaskByID(testDB, taskID)
+		require.NoError(t, err, "failed to fetch task")
+		numSecondsBefore := taskBefore.SecsSpent
+		taskLog, err := fetchTaskLogByID(testDB, tlID)
+		require.NoError(t, err, "failed to fetch task log")
+
+		// WHEN
+		err = DeleteTaskLogEntry(testDB, &taskLog)
+
+		// THEN
+		require.NoError(t, err, "failed to insert task log")
+
+		taskAfter, err := fetchTaskByID(testDB, taskID)
+		require.NoError(t, err, "failed to fetch task")
+
+		assert.Equal(t, numSecondsBefore-taskLog.SecsSpent, taskAfter.SecsSpent)
+	})
+
+	t.Run("TestFetchTLEntriesBetweenTS", func(t *testing.T) {
+		t.Cleanup(func() { cleanupDB(t, testDB) })
+
+		// GIVEN
+		referenceTS := time.Date(2024, time.September, 1, 9, 0, 0, 0, time.Local)
+		seedData := getTestData(referenceTS)
+		seedDB(t, testDB, seedData)
+
+		taskID := 1
+		numSeconds := 60 * 90
+		tlEndTS := referenceTS.Add(time.Hour * 2)
+		tlBeginTS := tlEndTS.Add(time.Second * -1 * time.Duration(numSeconds))
+		_, err = InsertManualTL(testDB, taskID, tlBeginTS, tlEndTS, taskLogComment)
+		require.NoError(t, err, "failed to insert task log")
+
+		// WHEN
+		reportBeginTS := referenceTS.Add(time.Hour * 24 * 7 * -2)
+		entries, err := FetchTLEntriesBetweenTS(testDB, reportBeginTS, referenceTS, 100)
+
+		// THEN
+		require.NoError(t, err, "failed to fetch report entries")
+		require.Len(t, entries, 3)
+	})
+
+	t.Run("TestFetchStats", func(t *testing.T) {
+		t.Cleanup(func() { cleanupDB(t, testDB) })
+
+		// GIVEN
+		referenceTS := time.Date(2024, time.September, 1, 9, 0, 0, 0, time.Local)
+		seedData := getTestData(referenceTS)
+		seedDB(t, testDB, seedData)
+
+		taskID := 1
+		comment := "an extra task log"
+		numSeconds := 60 * 90
+		tlEndTS := referenceTS.Add(time.Hour * 2)
+		tlBeginTS := tlEndTS.Add(time.Second * -1 * time.Duration(numSeconds))
+		_, err = InsertManualTL(testDB, taskID, tlBeginTS, tlEndTS, comment)
+		require.NoError(t, err, "failed to insert task log")
+
+		// WHEN
+		entries, err := FetchStats(testDB, 100)
+
+		// THEN
+		require.NoError(t, err, "failed to fetch report entries")
+		require.Len(t, entries, 2)
+
+		assert.Equal(t, 1, entries[0].TaskID)
+		assert.Equal(t, 3, entries[0].NumEntries)
+		assert.Equal(t, 5*secsInOneHour+numSeconds, entries[0].SecsSpent)
+
+		assert.Equal(t, 2, entries[1].TaskID)
+		assert.Equal(t, 1, entries[1].NumEntries)
+		assert.Equal(t, 4*secsInOneHour, entries[1].SecsSpent)
+	})
+
+	t.Run("TestFetchStatsBetweenTS", func(t *testing.T) {
+		t.Cleanup(func() { cleanupDB(t, testDB) })
+
+		// GIVEN
+		referenceTS := time.Date(2024, time.September, 1, 9, 0, 0, 0, time.Local)
+		seedData := getTestData(referenceTS)
+		seedDB(t, testDB, seedData)
+
+		taskID := 1
+		numSeconds := 60 * 90
+		tlEndTS := referenceTS.Add(time.Hour * 2)
+		tlBeginTS := tlEndTS.Add(time.Second * -1 * time.Duration(numSeconds))
+		_, err = InsertManualTL(testDB, taskID, tlBeginTS, tlEndTS, taskLogComment)
+		require.NoError(t, err, "failed to insert task log")
+
+		// WHEN
+		reportBeginTS := referenceTS.Add(time.Hour * 24 * 7 * -2)
+		entries, err := FetchStatsBetweenTS(testDB, reportBeginTS, referenceTS, 100)
+
+		// THEN
+		require.NoError(t, err, "failed to fetch report entries")
+		require.Len(t, entries, 2)
+
+		assert.Equal(t, 1, entries[0].TaskID)
+		assert.Equal(t, 2, entries[0].NumEntries)
+		assert.Equal(t, 5*secsInOneHour, entries[0].SecsSpent)
+
+		assert.Equal(t, 2, entries[1].TaskID)
+		assert.Equal(t, 1, entries[1].NumEntries)
+		assert.Equal(t, 4*secsInOneHour, entries[1].SecsSpent)
+	})
+
+	t.Run("TestFetchReportBetweenTS", func(t *testing.T) {
+		t.Cleanup(func() { cleanupDB(t, testDB) })
+
+		// GIVEN
+		referenceTS := time.Date(2024, time.September, 1, 9, 0, 0, 0, time.Local)
+		seedData := getTestData(referenceTS)
+		seedDB(t, testDB, seedData)
+
+		taskID := 1
+		numSeconds := 60 * 90
+		tlEndTS := referenceTS.Add(time.Hour * 2)
+		tlBeginTS := tlEndTS.Add(time.Second * -1 * time.Duration(numSeconds))
+		_, err = InsertManualTL(testDB, taskID, tlBeginTS, tlEndTS, taskLogComment)
+		require.NoError(t, err, "failed to insert task log")
+
+		// WHEN
+		reportBeginTS := referenceTS.Add(time.Hour * 24 * 7 * -2)
+		entries, err := FetchReportBetweenTS(testDB, reportBeginTS, referenceTS, 100)
+
+		// THEN
+		require.NoError(t, err, "failed to fetch report entries")
+
+		require.Len(t, entries, 2)
+		assert.Equal(t, 2, entries[0].TaskID)
+		assert.Equal(t, 1, entries[0].NumEntries)
+		assert.Equal(t, 4*secsInOneHour, entries[0].SecsSpent)
+
+		assert.Equal(t, 1, entries[1].TaskID)
+		assert.Equal(t, 2, entries[1].NumEntries)
+		assert.Equal(t, 5*secsInOneHour, entries[1].SecsSpent)
+	})
+
+	err = testDB.Close()
+	require.NoErrorf(t, err, "error closing DB: %v", err)
+}
+
+func cleanupDB(t *testing.T, testDB *sql.DB) {
+	t.Helper()
+
+	var err error
+	for _, tbl := range []string{"task_log", "task"} {
+		_, err = testDB.Exec(fmt.Sprintf("DELETE FROM %s", tbl))
+		require.NoErrorf(t, err, "failed to clean up table %q: %v", tbl, err)
+
+		_, err := testDB.Exec("DELETE FROM sqlite_sequence WHERE name=?;", tbl)
+		require.NoErrorf(t, err, "failed to reset auto increment for table %q: %v", tbl, err)
+	}
+}
+
+type testData struct {
+	tasks    []types.Task
+	taskLogs []types.TaskLogEntry
+}
+
+func getTestData(referenceTS time.Time) testData {
+	ua := referenceTS.UTC()
+	ca := ua.Add(time.Hour * 24 * 7 * -1)
+	tasks := []types.Task{
+		{
+			ID:        1,
+			Summary:   "seeded task 1",
+			Active:    true,
+			CreatedAt: ca,
+			UpdatedAt: ca.Add(time.Hour * 9),
+			SecsSpent: 5 * secsInOneHour,
+		},
+		{
+			ID:        2,
+			Summary:   "seeded task 2",
+			Active:    true,
+			CreatedAt: ca,
+			UpdatedAt: ca.Add(time.Hour * 6),
+			SecsSpent: 4 * secsInOneHour,
+		},
+	}
+
+	taskLogs := []types.TaskLogEntry{
+		{
+			ID:        1,
+			TaskID:    1,
+			BeginTS:   ca.Add(time.Hour * 2),
+			EndTS:     ca.Add(time.Hour * 4),
+			SecsSpent: 2 * secsInOneHour,
+			Comment:   "task 1 tl 1",
+		},
+		{
+			ID:        2,
+			TaskID:    1,
+			BeginTS:   ca.Add(time.Hour * 6),
+			EndTS:     ca.Add(time.Hour * 9),
+			SecsSpent: 3 * secsInOneHour,
+			Comment:   "task 1 tl 2",
+		},
+		{
+			ID:        3,
+			TaskID:    2,
+			BeginTS:   ca.Add(time.Hour * 2),
+			EndTS:     ca.Add(time.Hour * 6),
+			SecsSpent: 4 * secsInOneHour,
+			Comment:   "task 2 tl 1",
+		},
+	}
+
+	return testData{tasks, taskLogs}
+}
+
+func seedDB(t *testing.T, db *sql.DB, data testData) {
+	t.Helper()
+
+	for _, task := range data.tasks {
+		_, err := db.Exec(`
+INSERT INTO task (id, summary, secs_spent, active, created_at, updated_at)
+VALUES (?, ?, ?, ?, ?, ?)`, task.ID, task.Summary, task.SecsSpent, task.Active, task.CreatedAt, task.UpdatedAt)
+		require.NoError(t, err, "failed to insert data into table \"task\": %v", err)
+	}
+
+	for _, taskLog := range data.taskLogs {
+		_, err := db.Exec(`
+INSERT INTO task_log (id, task_id, begin_ts, end_ts, secs_spent, comment, active)
+VALUES (?, ?, ?, ?, ?, ?, ?)`, taskLog.ID, taskLog.TaskID, taskLog.BeginTS, taskLog.EndTS, taskLog.SecsSpent, taskLog.Comment, false)
+		require.NoError(t, err, "failed to insert data into table \"task_log\": %v", err)
+	}
+}
diff --git a/internal/ui/date_helpers.go b/internal/types/date_helpers.go
similarity index 54%
rename from internal/ui/date_helpers.go
rename to internal/types/date_helpers.go
index 62abf24..ee6293e 100644
--- a/internal/ui/date_helpers.go
+++ b/internal/types/date_helpers.go
@@ -1,63 +1,74 @@
-package ui
+package types
 
 import (
+	"errors"
 	"fmt"
 	"strings"
 	"time"
 )
 
 const (
 	timePeriodDaysUpperBound = 7
+	TimePeriodWeek           = "week"
+	timeFormat               = "2006/01/02 15:04"
+	timeOnlyFormat           = "15:04"
+	dayFormat                = "Monday"
+	friendlyTimeFormat       = "Mon, 15:04"
+	dateFormat               = "2006/01/02"
 )
 
 var (
-	timePeriodNotValidErr = fmt.Errorf("time period is not valid; accepted values: day, yest, week, 3d, date (eg. %s), or date range (eg. %s...%s)", dateFormat, dateFormat, dateFormat)
-	timePeriodTooLargeErr = fmt.Errorf("time period is too large; maximum number of days allowed (both inclusive): %d", timePeriodDaysUpperBound)
+	errDateRangeIncorrect         = errors.New("date range is incorrect")
+	errStartDateIncorrect         = errors.New("start date is incorrect")
+	errEndDateIncorrect           = errors.New("end date is incorrect")
+	errEndDateIsNotAfterStartDate = errors.New("end date is not after start date")
+	errTimePeriodNotValid         = errors.New("time period is not valid")
+	errTimePeriodTooLarge         = errors.New("time period is too large")
 )
 
-type timePeriod struct {
-	start   time.Time
-	end     time.Time
-	numDays int
+type TimePeriod struct {
+	Start   time.Time
+	End     time.Time
+	NumDays int
 }
 
-func parseDateDuration(dateRange string) (timePeriod, bool) {
-	var tp timePeriod
+func parseDateDuration(dateRange string) (TimePeriod, error) {
+	var tp TimePeriod
 
 	elements := strings.Split(dateRange, "...")
 	if len(elements) != 2 {
-		return tp, false
+		return tp, fmt.Errorf("%w: date range needs to be of the format: %s...%s", errDateRangeIncorrect, dateFormat, dateFormat)
 	}
 
 	start, err := time.ParseInLocation(string(dateFormat), elements[0], time.Local)
 	if err != nil {
-		return tp, false
+		return tp, fmt.Errorf("%w: %s", errStartDateIncorrect, err.Error())
 	}
 
 	end, err := time.ParseInLocation(string(dateFormat), elements[1], time.Local)
 	if err != nil {
-		return tp, false
+		return tp, fmt.Errorf("%w: %s", errEndDateIncorrect, err.Error())
 	}
 
 	if end.Sub(start) <= 0 {
-		return tp, false
+		return tp, fmt.Errorf("%w", errEndDateIsNotAfterStartDate)
 	}
 
-	tp.start = start
-	tp.end = end
-	tp.numDays = int(end.Sub(start).Hours()/24) + 1
+	tp.Start = start
+	tp.End = end
+	tp.NumDays = int(end.Sub(start).Hours()/24) + 1
 
-	return tp, true
+	return tp, nil
 }
 
-func getTimePeriod(period string, now time.Time, fullWeek bool) (timePeriod, error) {
+func GetTimePeriod(period string, now time.Time, fullWeek bool) (TimePeriod, error) {
 	var start, end time.Time
 	var numDays int
 
 	switch period {
 
 	case "today":
 		start = time.Date(now.Year(), now.Month(), now.Day(), 0, 0, 0, 0, now.Location())
 		end = start.AddDate(0, 0, 1)
 		numDays = 1
 
@@ -68,94 +79,79 @@ func getTimePeriod(period string, now time.Time, fullWeek bool) (timePeriod, err
 		end = start.AddDate(0, 0, 1)
 		numDays = 1
 
 	case "3d":
 		threeDaysBefore := now.AddDate(0, 0, -2)
 
 		start = time.Date(threeDaysBefore.Year(), threeDaysBefore.Month(), threeDaysBefore.Day(), 0, 0, 0, 0, threeDaysBefore.Location())
 		end = start.AddDate(0, 0, 3)
 		numDays = 3
 
-	case "week":
+	case TimePeriodWeek:
 		weekday := now.Weekday()
 		offset := (7 + weekday - time.Monday) % 7
 		startOfWeek := now.AddDate(0, 0, -int(offset))
 		start = time.Date(startOfWeek.Year(), startOfWeek.Month(), startOfWeek.Day(), 0, 0, 0, 0, startOfWeek.Location())
 		if fullWeek {
 			numDays = 7
 		} else {
 			numDays = int(offset) + 1
 		}
 		end = start.AddDate(0, 0, numDays)
 
 	default:
 		var err error
 
 		if strings.Contains(period, "...") {
-			var ts timePeriod
-			var ok bool
-			ts, ok = parseDateDuration(period)
-			if !ok {
-				return ts, timePeriodNotValidErr
-			}
-			if ts.numDays > timePeriodDaysUpperBound {
-				return ts, timePeriodTooLargeErr
+			var ts TimePeriod
+			ts, err = parseDateDuration(period)
+			if err != nil {
+				return ts, fmt.Errorf("%w: %s", errTimePeriodNotValid, err.Error())
 			}
 
-			start = ts.start
-			end = ts.end.AddDate(0, 0, 1)
-			numDays = ts.numDays
+			if ts.NumDays > timePeriodDaysUpperBound {
+				return ts, fmt.Errorf("%w: maximum number of days allowed (both inclusive): %d", errTimePeriodTooLarge, timePeriodDaysUpperBound)
+			}
+
+			start = ts.Start
+			end = ts.End.AddDate(0, 0, 1)
+			numDays = ts.NumDays
 		} else {
 			start, err = time.ParseInLocation(string(dateFormat), period, time.Local)
 			if err != nil {
-				return timePeriod{}, timePeriodNotValidErr
+				return TimePeriod{}, fmt.Errorf("%w: %s", errTimePeriodNotValid, err.Error())
 			}
 			end = start.AddDate(0, 0, 1)
 			numDays = 1
 		}
 	}
 
-	return timePeriod{
-		start:   start,
-		end:     end,
-		numDays: numDays,
+	return TimePeriod{
+		Start:   start,
+		End:     end,
+		NumDays: numDays,
 	}, nil
 }
 
-type timeShiftDirection uint8
-
-const (
-	shiftForward timeShiftDirection = iota
-	shiftBackward
-)
-
-type timeShiftDuration uint8
-
-const (
-	shiftMinute timeShiftDuration = iota
-	shiftFiveMinutes
-	shiftHour
-)
-
-func getShiftedTime(ts time.Time, direction timeShiftDirection, duration timeShiftDuration) time.Time {
+func GetShiftedTime(ts time.Time, direction TimeShiftDirection, duration TimeShiftDuration) time.Time {
 	var d time.Duration
 
 	switch duration {
-	case shiftMinute:
+	case ShiftMinute:
 		d = time.Minute
-	case shiftFiveMinutes:
+	case ShiftFiveMinutes:
 		d = time.Minute * 5
-	case shiftHour:
+	case ShiftHour:
 		d = time.Hour
 	}
 
-	if direction == shiftBackward {
+	if direction == ShiftBackward {
 		d = -1 * d
 	}
 	return ts.Add(d)
 }
 
 type tsRelative uint8
 
 const (
 	tsFromFuture tsRelative = iota
 	tsFromToday
diff --git a/internal/ui/date_helpers_test.go b/internal/types/date_helpers_test.go
similarity index 84%
rename from internal/ui/date_helpers_test.go
rename to internal/types/date_helpers_test.go
index df3fbd7..8cd1bc3 100644
--- a/internal/ui/date_helpers_test.go
+++ b/internal/types/date_helpers_test.go
@@ -1,95 +1,100 @@
-package ui
+package types
 
 import (
 	"testing"
 	"time"
 
 	"github.com/stretchr/testify/assert"
+	"github.com/stretchr/testify/require"
 )
 
 func TestParseDateDuration(t *testing.T) {
 	testCases := []struct {
 		name             string
 		input            string
 		expectedStartStr string
 		expectedEndStr   string
 		expectedNumDays  int
-		ok               bool
+		err              error
 	}{
 		// success
 		{
 			name:             "a range of 1 day",
 			input:            "2024/06/10...2024/06/11",
 			expectedStartStr: "2024/06/10 00:00",
 			expectedEndStr:   "2024/06/11 00:00",
 			expectedNumDays:  2,
-			ok:               true,
 		},
 		{
 			name:             "a range of 2 days",
 			input:            "2024/06/29...2024/07/01",
 			expectedStartStr: "2024/06/29 00:00",
 			expectedEndStr:   "2024/07/01 00:00",
 			expectedNumDays:  3,
-			ok:               true,
 		},
 		{
 			name:             "a range of 1 year",
 			input:            "2024/06/29...2025/06/29",
 			expectedStartStr: "2024/06/29 00:00",
 			expectedEndStr:   "2025/06/29 00:00",
 			expectedNumDays:  366,
-			ok:               true,
 		},
 		// failures
 		{
 			name:  "empty string",
 			input: "",
+			err:   errDateRangeIncorrect,
 		},
 		{
 			name:  "only one date",
 			input: "2024/06/10",
+			err:   errDateRangeIncorrect,
 		},
 		{
 			name:  "badly formatted start date",
 			input: "2024/0610...2024/06/10",
+			err:   errStartDateIncorrect,
 		},
 		{
 			name:  "badly formatted end date",
 			input: "2024/06/10...2024/0610",
+			err:   errEndDateIncorrect,
 		},
 		{
 			name:  "a range of 0 days",
 			input: "2024/06/10...2024/06/10",
+			err:   errEndDateIsNotAfterStartDate,
 		},
 		{
 			name:  "end date before start date",
 			input: "2024/06/10...2024/06/08",
+			err:   errEndDateIsNotAfterStartDate,
 		},
 	}
 
 	for _, tt := range testCases {
 		t.Run(tt.name, func(t *testing.T) {
-			got, ok := parseDateDuration(tt.input)
+			got, err := parseDateDuration(tt.input)
 
-			if tt.ok {
-				startStr := got.start.Format(timeFormat)
-				endStr := got.end.Format(timeFormat)
-
-				assert.True(t, ok)
-				assert.Equal(t, tt.expectedStartStr, startStr)
-				assert.Equal(t, tt.expectedEndStr, endStr)
-				assert.Equal(t, tt.expectedNumDays, got.numDays)
-			} else {
-				assert.False(t, ok)
+			if tt.err != nil {
+				assert.ErrorIs(t, err, tt.err)
+				return
 			}
+
+			startStr := got.Start.Format(timeFormat)
+			endStr := got.End.Format(timeFormat)
+
+			require.NoError(t, err)
+			assert.Equal(t, tt.expectedStartStr, startStr)
+			assert.Equal(t, tt.expectedEndStr, endStr)
+			assert.Equal(t, tt.expectedNumDays, got.NumDays)
 		})
 	}
 }
 
 func TestGetTimePeriod(t *testing.T) {
 	now, err := time.ParseInLocation(string(timeFormat), "2024/06/20 20:00", time.Local)
 	if err != nil {
 		t.Fatalf("error setting up the test: time is not valid: %s", err)
 	}
 
@@ -174,49 +179,49 @@ func TestGetTimePeriod(t *testing.T) {
 			name:             "a date range",
 			period:           "2024/06/15...2024/06/20",
 			expectedStartStr: "2024/06/15 00:00",
 			expectedEndStr:   "2024/06/21 00:00",
 			expectedNumDays:  6,
 		},
 		// failures
 		{
 			name:   "a faulty date",
 			period: "2024/06-15",
-			err:    timePeriodNotValidErr,
+			err:    errTimePeriodNotValid,
 		},
 		{
 			name:   "a faulty date range",
 			period: "2024/06/15...2024",
-			err:    timePeriodNotValidErr,
+			err:    errTimePeriodNotValid,
 		},
 		{
 			name:   "a date range too large",
 			period: "2024/06/15...2024/06/22",
-			err:    timePeriodTooLargeErr,
+			err:    errTimePeriodTooLarge,
 		},
 	}
 
 	for _, tt := range testCases {
 		t.Run(tt.name, func(t *testing.T) {
-			got, err := getTimePeriod(tt.period, tt.now, tt.fullWeek)
+			got, err := GetTimePeriod(tt.period, tt.now, tt.fullWeek)
 
-			startStr := got.start.Format(timeFormat)
-			endStr := got.end.Format(timeFormat)
+			startStr := got.Start.Format(timeFormat)
+			endStr := got.End.Format(timeFormat)
 
 			if tt.err == nil {
 				assert.Equal(t, tt.expectedStartStr, startStr)
 				assert.Equal(t, tt.expectedEndStr, endStr)
-				assert.Equal(t, tt.expectedNumDays, got.numDays)
-				assert.Nil(t, err)
-			} else {
-				assert.Equal(t, tt.err, err)
+				assert.Equal(t, tt.expectedNumDays, got.NumDays)
+				assert.NoError(t, err)
+				return
 			}
+			assert.ErrorIs(t, err, tt.err)
 		})
 	}
 }
 
 func TestGetTSRelative(t *testing.T) {
 	reference := time.Date(2024, 6, 29, 12, 0, 0, 0, time.Local)
 	testCases := []struct {
 		name      string
 		ts        time.Time
 		reference time.Time
diff --git a/internal/types/types.go b/internal/types/types.go
new file mode 100644
index 0000000..b34a7ad
--- /dev/null
+++ b/internal/types/types.go
@@ -0,0 +1,158 @@
+package types
+
+import (
+	"fmt"
+	"math"
+	"time"
+
+	"github.com/dhth/hours/internal/utils"
+	"github.com/dustin/go-humanize"
+)
+
+type Task struct {
+	ID             int
+	Summary        string
+	CreatedAt      time.Time
+	UpdatedAt      time.Time
+	TrackingActive bool
+	SecsSpent      int
+	Active         bool
+	TaskTitle      string
+	TaskDesc       string
+}
+
+type TaskLogEntry struct {
+	ID          int
+	TaskID      int
+	TaskSummary string
+	BeginTS     time.Time
+	EndTS       time.Time
+	SecsSpent   int
+	Comment     string
+	TLTitle     string
+	TLDesc      string
+}
+
+type ActiveTaskDetails struct {
+	TaskID              int
+	TaskSummary         string
+	LastLogEntryBeginTS time.Time
+}
+
+type TaskReportEntry struct {
+	TaskID      int
+	TaskSummary string
+	NumEntries  int
+	SecsSpent   int
+}
+
+func (t *Task) UpdateTitle() {
+	var trackingIndicator string
+	if t.TrackingActive {
+		trackingIndicator = "⏲ "
+	}
+
+	t.TaskTitle = trackingIndicator + t.Summary
+}
+
+func (t *Task) UpdateDesc() {
+	var timeSpent string
+
+	if t.SecsSpent != 0 {
+		timeSpent = "worked on for " + HumanizeDuration(t.SecsSpent)
+	} else {
+		timeSpent = "no time spent"
+	}
+	lastUpdated := fmt.Sprintf("last updated: %s", humanize.Time(t.UpdatedAt))
+
+	t.TaskDesc = fmt.Sprintf("%s %s", utils.RightPadTrim(lastUpdated, 60, true), timeSpent)
+}
+
+func (tl *TaskLogEntry) UpdateTitle() {
+	tl.TLTitle = utils.Trim(tl.Comment, 60)
+}
+
+func (tl *TaskLogEntry) UpdateDesc() {
+	timeSpentStr := HumanizeDuration(tl.SecsSpent)
+
+	var timeStr string
+	var durationMsg string
+
+	endTSRelative := getTSRelative(tl.EndTS, time.Now())
+
+	switch endTSRelative {
+	case tsFromToday:
+		durationMsg = fmt.Sprintf("%s  ...  %s", tl.BeginTS.Format(timeOnlyFormat), tl.EndTS.Format(timeOnlyFormat))
+	case tsFromYesterday:
+		durationMsg = "Yesterday"
+	case tsFromThisWeek:
+		durationMsg = tl.EndTS.Format(dayFormat)
+	default:
+		durationMsg = humanize.Time(tl.EndTS)
+	}
+
+	timeStr = fmt.Sprintf("%s (%s)",
+		utils.RightPadTrim(durationMsg, 40, true),
+		timeSpentStr)
+
+	tl.TLDesc = fmt.Sprintf("%s %s", utils.RightPadTrim("["+tl.TaskSummary+"]", 60, true), timeStr)
+}
+
+func (t Task) Title() string {
+	return t.TaskTitle
+}
+
+func (t Task) Description() string {
+	return t.TaskDesc
+}
+
+func (t Task) FilterValue() string {
+	return t.Summary
+}
+
+func (tl TaskLogEntry) Title() string {
+	return tl.TLTitle
+}
+
+func (tl TaskLogEntry) Description() string {
+	return tl.TLDesc
+}
+
+func (tl TaskLogEntry) FilterValue() string {
+	return tl.Comment
+}
+
+func HumanizeDuration(durationInSecs int) string {
+	duration := time.Duration(durationInSecs) * time.Second
+
+	if duration.Seconds() < 60 {
+		return fmt.Sprintf("%ds", int(duration.Seconds()))
+	}
+
+	if duration.Minutes() < 60 {
+		return fmt.Sprintf("%dm", int(duration.Minutes()))
+	}
+
+	modMins := int(math.Mod(duration.Minutes(), 60))
+
+	if modMins == 0 {
+		return fmt.Sprintf("%dh", int(duration.Hours()))
+	}
+
+	return fmt.Sprintf("%dh %dm", int(duration.Hours()), modMins)
+}
+
+type TimeShiftDirection uint8
+
+const (
+	ShiftForward TimeShiftDirection = iota
+	ShiftBackward
+)
+
+type TimeShiftDuration uint8
+
+const (
+	ShiftMinute TimeShiftDuration = iota
+	ShiftFiveMinutes
+	ShiftHour
+)
diff --git a/internal/types/types_test.go b/internal/types/types_test.go
new file mode 100644
index 0000000..8fc09b3
--- /dev/null
+++ b/internal/types/types_test.go
@@ -0,0 +1,58 @@
+package types
+
+import (
+	"testing"
+
+	"github.com/stretchr/testify/assert"
+)
+
+func TestHumanizeDuration(t *testing.T) {
+	testCases := []struct {
+		name     string
+		input    int
+		expected string
+	}{
+		{
+			name:     "0 seconds",
+			input:    0,
+			expected: "0s",
+		},
+		{
+			name:     "30 seconds",
+			input:    30,
+			expected: "30s",
+		},
+		{
+			name:     "60 seconds",
+			input:    60,
+			expected: "1m",
+		},
+		{
+			name:     "1805 seconds",
+			input:    1805,
+			expected: "30m",
+		},
+		{
+			name:     "3605 seconds",
+			input:    3605,
+			expected: "1h",
+		},
+		{
+			name:     "4200 seconds",
+			input:    4200,
+			expected: "1h 10m",
+		},
+		{
+			name:     "87000 seconds",
+			input:    87000,
+			expected: "24h 10m",
+		},
+	}
+
+	for _, tt := range testCases {
+		t.Run(tt.name, func(t *testing.T) {
+			got := HumanizeDuration(tt.input)
+			assert.Equal(t, tt.expected, got)
+		})
+	}
+}
diff --git a/internal/ui/active.go b/internal/ui/active.go
index bb5a35d..5cf26f6 100644
--- a/internal/ui/active.go
+++ b/internal/ui/active.go
@@ -1,42 +1,43 @@
 package ui
 
 import (
 	"database/sql"
 	"fmt"
 	"io"
-	"os"
 	"strings"
 	"time"
+
+	pers "github.com/dhth/hours/internal/persistence"
+	"github.com/dhth/hours/internal/types"
 )
 
 const (
 	ActiveTaskPlaceholder     = "{{task}}"
 	ActiveTaskTimePlaceholder = "{{time}}"
 	activeSecsThreshold       = 60
 	activeSecsThresholdStr    = "<1m"
 )
 
-func ShowActiveTask(db *sql.DB, writer io.Writer, template string) {
-	activeTaskDetails, err := fetchActiveTaskFromDB(db)
+func ShowActiveTask(db *sql.DB, writer io.Writer, template string) error {
+	activeTaskDetails, err := pers.FetchActiveTask(db)
 	if err != nil {
-		fmt.Fprintf(os.Stdout, "Something went wrong:\n%s", err)
-		os.Exit(1)
+		return err
 	}
 
-	if activeTaskDetails.taskId == -1 {
-		return
+	if activeTaskDetails.TaskID == -1 {
+		return nil
 	}
 
-	now := time.Now()
-	timeSpent := now.Sub(activeTaskDetails.lastLogEntryBeginTs).Seconds()
+	timeSpent := time.Since(activeTaskDetails.LastLogEntryBeginTS).Seconds()
 	var timeSpentStr string
 	if timeSpent <= activeSecsThreshold {
 		timeSpentStr = activeSecsThresholdStr
 	} else {
-		timeSpentStr = humanizeDuration(int(timeSpent))
+		timeSpentStr = types.HumanizeDuration(int(timeSpent))
 	}
 
-	activeStr := strings.Replace(template, ActiveTaskPlaceholder, activeTaskDetails.taskSummary, 1)
+	activeStr := strings.Replace(template, ActiveTaskPlaceholder, activeTaskDetails.TaskSummary, 1)
 	activeStr = strings.Replace(activeStr, ActiveTaskTimePlaceholder, timeSpentStr, 1)
 	fmt.Fprint(writer, activeStr)
+	return nil
 }
diff --git a/internal/ui/cmds.go b/internal/ui/cmds.go
index b4d7c1b..020d771 100644
--- a/internal/ui/cmds.go
+++ b/internal/ui/cmds.go
@@ -1,161 +1,164 @@
 package ui
 
 import (
 	"database/sql"
+	"errors"
 	"time"
 
 	tea "github.com/charmbracelet/bubbletea"
-	_ "modernc.org/sqlite"
+	pers "github.com/dhth/hours/internal/persistence"
+	"github.com/dhth/hours/internal/types"
+	_ "modernc.org/sqlite" // sqlite driver
 )
 
 func toggleTracking(db *sql.DB,
-	taskId int,
+	taskID int,
 	beginTs time.Time,
 	endTs time.Time,
 	comment string,
 ) tea.Cmd {
 	return func() tea.Msg {
 		row := db.QueryRow(`
 SELECT id, task_id
 FROM task_log
 WHERE active=1
 ORDER BY begin_ts DESC
 LIMIT 1
 `)
 		var trackStatus trackingStatus
-		var activeTaskLogId int
-		var activeTaskId int
+		var activeTaskLogID int
+		var activeTaskID int
 
-		err := row.Scan(&activeTaskLogId, &activeTaskId)
-		if err == sql.ErrNoRows {
+		err := row.Scan(&activeTaskLogID, &activeTaskID)
+		if errors.Is(err, sql.ErrNoRows) {
 			trackStatus = trackingInactive
 		} else if err != nil {
 			return trackingToggledMsg{err: err}
 		} else {
 			trackStatus = trackingActive
 		}
 
 		switch trackStatus {
 		case trackingInactive:
-			err = insertNewTLInDB(db, taskId, beginTs)
+			_, err = pers.InsertNewTL(db, taskID, beginTs)
 			if err != nil {
 				return trackingToggledMsg{err: err}
 			} else {
-				return trackingToggledMsg{taskId: taskId}
+				return trackingToggledMsg{taskID: taskID}
 			}
 
 		default:
 			secsSpent := int(endTs.Sub(beginTs).Seconds())
-			err := updateActiveTLInDB(db, activeTaskLogId, activeTaskId, beginTs, endTs, secsSpent, comment)
+			err := pers.UpdateActiveTL(db, activeTaskLogID, activeTaskID, beginTs, endTs, secsSpent, comment)
 			if err != nil {
 				return trackingToggledMsg{err: err}
 			} else {
-				return trackingToggledMsg{taskId: taskId, finished: true, secsSpent: secsSpent}
+				return trackingToggledMsg{taskID: taskID, finished: true, secsSpent: secsSpent}
 			}
 		}
 	}
 }
 
 func updateTLBeginTS(db *sql.DB, beginTS time.Time) tea.Cmd {
 	return func() tea.Msg {
-		err := updateTLBeginTSInDB(db, beginTS)
+		err := pers.UpdateTLBeginTS(db, beginTS)
 		return tlBeginTSUpdatedMsg{beginTS, err}
 	}
 }
 
-func insertManualEntry(db *sql.DB, taskId int, beginTS time.Time, endTS time.Time, comment string) tea.Cmd {
+func insertManualEntry(db *sql.DB, taskID int, beginTS time.Time, endTS time.Time, comment string) tea.Cmd {
 	return func() tea.Msg {
-		err := insertManualTLInDB(db, taskId, beginTS, endTS, comment)
-		return manualTaskLogInserted{taskId, err}
+		_, err := pers.InsertManualTL(db, taskID, beginTS, endTS, comment)
+		return manualTaskLogInserted{taskID, err}
 	}
 }
 
 func fetchActiveTask(db *sql.DB) tea.Cmd {
 	return func() tea.Msg {
-		activeTaskDetails, err := fetchActiveTaskFromDB(db)
+		activeTaskDetails, err := pers.FetchActiveTask(db)
 		if err != nil {
 			return activeTaskFetchedMsg{err: err}
 		}
 
-		if activeTaskDetails.taskId == -1 {
+		if activeTaskDetails.TaskID == -1 {
 			return activeTaskFetchedMsg{noneActive: true}
 		}
 
 		return activeTaskFetchedMsg{
-			activeTaskId: activeTaskDetails.taskId,
-			beginTs:      activeTaskDetails.lastLogEntryBeginTs,
+			activeTaskID: activeTaskDetails.TaskID,
+			beginTs:      activeTaskDetails.LastLogEntryBeginTS,
 		}
 	}
 }
 
-func updateTaskRep(db *sql.DB, t *task) tea.Cmd {
+func updateTaskRep(db *sql.DB, t *types.Task) tea.Cmd {
 	return func() tea.Msg {
-		err := updateTaskDataFromDB(db, t)
+		err := pers.UpdateTaskData(db, t)
 		return taskRepUpdatedMsg{
 			tsk: t,
 			err: err,
 		}
 	}
 }
 
 func fetchTaskLogEntries(db *sql.DB) tea.Cmd {
 	return func() tea.Msg {
-		entries, err := fetchTLEntriesFromDB(db, true, 50)
+		entries, err := pers.FetchTLEntries(db, true, 50)
 		return taskLogEntriesFetchedMsg{
 			entries: entries,
 			err:     err,
 		}
 	}
 }
 
-func deleteLogEntry(db *sql.DB, entry *taskLogEntry) tea.Cmd {
+func deleteLogEntry(db *sql.DB, entry *types.TaskLogEntry) tea.Cmd {
 	return func() tea.Msg {
-		err := deleteEntry(db, entry)
+		err := pers.DeleteTaskLogEntry(db, entry)
 		return taskLogEntryDeletedMsg{
 			entry: entry,
 			err:   err,
 		}
 	}
 }
 
 func deleteActiveTaskLog(db *sql.DB) tea.Cmd {
 	return func() tea.Msg {
-		err := deleteActiveTLInDB(db)
+		err := pers.DeleteActiveTL(db)
 		return activeTaskLogDeletedMsg{err}
 	}
 }
 
 func createTask(db *sql.DB, summary string) tea.Cmd {
 	return func() tea.Msg {
-		err := insertTaskInDB(db, summary)
+		_, err := pers.InsertTask(db, summary)
 		return taskCreatedMsg{err}
 	}
 }
 
-func updateTask(db *sql.DB, task *task, summary string) tea.Cmd {
+func updateTask(db *sql.DB, task *types.Task, summary string) tea.Cmd {
 	return func() tea.Msg {
-		err := updateTaskInDB(db, task.id, summary)
+		err := pers.UpdateTask(db, task.ID, summary)
 		return taskUpdatedMsg{task, summary, err}
 	}
 }
 
-func updateTaskActiveStatus(db *sql.DB, task *task, active bool) tea.Cmd {
+func updateTaskActiveStatus(db *sql.DB, task *types.Task, active bool) tea.Cmd {
 	return func() tea.Msg {
-		err := updateTaskActiveStatusInDB(db, task.id, active)
+		err := pers.UpdateTaskActiveStatus(db, task.ID, active)
 		return taskActiveStatusUpdated{task, active, err}
 	}
 }
 
 func fetchTasks(db *sql.DB, active bool) tea.Cmd {
 	return func() tea.Msg {
-		tasks, err := fetchTasksFromDB(db, active, 50)
+		tasks, err := pers.FetchTasks(db, active, 50)
 		return tasksFetched{tasks, active, err}
 	}
 }
 
 func hideHelp(interval time.Duration) tea.Cmd {
 	return tea.Tick(interval, func(time.Time) tea.Msg {
 		return HideHelpMsg{}
 	})
 }
 
@@ -163,23 +166,23 @@ func getRecordsData(analyticsType recordsType, db *sql.DB, period string, start,
 	return func() tea.Msg {
 		var data string
 		var err error
 
 		switch analyticsType {
 		case reportRecords:
 			data, err = getReport(db, start, numDays, plain)
 		case reportAggRecords:
 			data, err = getReportAgg(db, start, numDays, plain)
 		case reportLogs:
-			data, err = renderTaskLog(db, start, end, 20, plain)
+			data, err = getTaskLog(db, start, end, 20, plain)
 		case reportStats:
-			data, err = renderStats(db, period, start, end, plain)
+			data, err = getStats(db, period, start, end, plain)
 		}
 
 		return recordsDataFetchedMsg{
 			start:  start,
 			end:    end,
 			report: data,
 			err:    err,
 		}
 	}
 }
diff --git a/internal/ui/generate.go b/internal/ui/generate.go
index 36957cb..8788289 100644
--- a/internal/ui/generate.go
+++ b/internal/ui/generate.go
@@ -1,17 +1,19 @@
 package ui
 
 import (
 	"database/sql"
 	"fmt"
 	"math/rand"
 	"time"
+
+	pers "github.com/dhth/hours/internal/persistence"
 )
 
 var (
 	tasks = []string{
 		".net",
 		"assembly",
 		"c",
 		"c#",
 		"c++",
 		"clojure",
@@ -85,31 +87,31 @@ var (
 		"report",
 		"script",
 		"workflow",
 		"log",
 	}
 )
 
 func GenerateData(db *sql.DB, numDays, numTasks uint8) error {
 	for i := uint8(0); i < numTasks; i++ {
 		summary := tasks[rand.Intn(len(tasks))]
-		err := insertTaskInDB(db, summary)
+		_, err := pers.InsertTask(db, summary)
 		if err != nil {
 			return err
 		}
 		numLogs := int(numDays/2) + rand.Intn(int(numDays/2))
 		for j := 0; j < numLogs; j++ {
 			beginTs := randomTimestamp(int(numDays))
 			numMinutes := 30 + rand.Intn(60)
 			endTs := beginTs.Add(time.Minute * time.Duration(numMinutes))
 			comment := fmt.Sprintf("%s %s", verbs[rand.Intn(len(verbs))], nouns[rand.Intn(len(nouns))])
-			err = insertManualTLInDB(db, int(i+1), beginTs, endTs, comment)
+			_, err = pers.InsertManualTL(db, int(i+1), beginTs, endTs, comment)
 			if err != nil {
 				return err
 			}
 		}
 	}
 
 	return nil
 }
 
 func randomTimestamp(numDays int) time.Time {
diff --git a/internal/ui/initial.go b/internal/ui/initial.go
index ab83b95..783a5a6 100644
--- a/internal/ui/initial.go
+++ b/internal/ui/initial.go
@@ -1,22 +1,23 @@
 package ui
 
 import (
 	"database/sql"
 	"time"
 
 	"github.com/charmbracelet/bubbles/list"
 	"github.com/charmbracelet/bubbles/textinput"
 	"github.com/charmbracelet/lipgloss"
+	"github.com/dhth/hours/internal/types"
 )
 
-func InitialModel(db *sql.DB) model {
+func InitialModel(db *sql.DB) Model {
 	var activeTaskItems []list.Item
 	var inactiveTaskItems []list.Item
 	var tasklogListItems []list.Item
 
 	trackingInputs := make([]textinput.Model, 3)
 	trackingInputs[entryBeginTS] = textinput.New()
 	trackingInputs[entryBeginTS].Placeholder = "09:30"
 	trackingInputs[entryBeginTS].Focus()
 	trackingInputs[entryBeginTS].CharLimit = len(string(timeFormat))
 	trackingInputs[entryBeginTS].Width = 30
@@ -33,25 +34,25 @@ func InitialModel(db *sql.DB) model {
 	trackingInputs[entryComment].CharLimit = 255
 	trackingInputs[entryComment].Width = 80
 
 	taskInputs := make([]textinput.Model, 3)
 	taskInputs[summaryField] = textinput.New()
 	taskInputs[summaryField].Placeholder = "task summary goes here"
 	taskInputs[summaryField].Focus()
 	taskInputs[summaryField].CharLimit = 100
 	taskInputs[entryBeginTS].Width = 60
 
-	m := model{
+	m := Model{
 		db:                 db,
 		activeTasksList:    list.New(activeTaskItems, newItemDelegate(lipgloss.Color(activeTaskListColor)), listWidth, 0),
 		inactiveTasksList:  list.New(inactiveTaskItems, newItemDelegate(lipgloss.Color(inactiveTaskListColor)), listWidth, 0),
-		activeTaskMap:      make(map[int]*task),
+		activeTaskMap:      make(map[int]*types.Task),
 		activeTaskIndexMap: make(map[int]int),
 		taskLogList:        list.New(tasklogListItems, newItemDelegate(lipgloss.Color(taskLogListColor)), listWidth, 0),
 		showHelpIndicator:  true,
 		trackingInputs:     trackingInputs,
 		taskInputs:         taskInputs,
 	}
 	m.activeTasksList.Title = "Tasks"
 	m.activeTasksList.SetStatusBarItemName("task", "tasks")
 	m.activeTasksList.DisableQuitKeybindings()
 	m.activeTasksList.SetShowHelp(false)
diff --git a/internal/ui/log.go b/internal/ui/log.go
index d4fcb7a..50b4de9 100644
--- a/internal/ui/log.go
+++ b/internal/ui/log.go
@@ -1,113 +1,121 @@
 package ui
 
 import (
 	"bytes"
 	"database/sql"
+	"errors"
 	"fmt"
 	"io"
-	"os"
 	"time"
 
 	tea "github.com/charmbracelet/bubbletea"
 	"github.com/charmbracelet/lipgloss"
+	pers "github.com/dhth/hours/internal/persistence"
+	"github.com/dhth/hours/internal/types"
+	"github.com/dhth/hours/internal/utils"
 	"github.com/olekukonko/tablewriter"
 )
 
 const (
-	logNumDaysUpperBound = 7
-	logTimeCharsBudget   = 6
+	logNumDaysUpperBound   = 7
+	logTimeCharsBudget     = 6
+	interactiveLogDayLimit = 1
 )
 
-func RenderTaskLog(db *sql.DB, writer io.Writer, plain bool, period string, interactive bool) {
+var (
+	errInteractiveModeNotApplicable = errors.New("interactive mode is not applicable")
+	errCouldntGenerateLogs          = errors.New("couldn't generate logs")
+)
+
+func RenderTaskLog(db *sql.DB, writer io.Writer, plain bool, period string, interactive bool) error {
 	if period == "" {
-		return
+		return nil
 	}
 
-	ts, err := getTimePeriod(period, time.Now(), false)
+	ts, err := types.GetTimePeriod(period, time.Now(), false)
 	if err != nil {
-		fmt.Printf("error: %s\n", err)
-		os.Exit(1)
+		return err
 	}
 
-	if interactive && ts.numDays > 1 {
-		fmt.Print("Interactive mode for logs is limited to a day; use non-interactive mode to see logs for a larger time period\n")
-		os.Exit(1)
+	if interactive && ts.NumDays > interactiveLogDayLimit {
+		return fmt.Errorf("%w (limited to %d day); use non-interactive mode to see logs for a larger time period", errInteractiveModeNotApplicable, interactiveLogDayLimit)
 	}
 
-	log, err := renderTaskLog(db, ts.start, ts.end, 100, plain)
+	log, err := getTaskLog(db, ts.Start, ts.End, 100, plain)
 	if err != nil {
-		fmt.Printf("Something went wrong generating the log: %s\n", err)
+		return fmt.Errorf("%w: %s", errCouldntGenerateLogs, err.Error())
 	}
 
 	if interactive {
-		p := tea.NewProgram(initialRecordsModel(reportLogs, db, ts.start, ts.end, plain, period, ts.numDays, log))
-		if _, err := p.Run(); err != nil {
-			fmt.Printf("Alas, there has been an error: %v", err)
-			os.Exit(1)
+		p := tea.NewProgram(initialRecordsModel(reportLogs, db, ts.Start, ts.End, plain, period, ts.NumDays, log))
+		_, err := p.Run()
+		if err != nil {
+			return err
 		}
 	} else {
 		fmt.Fprint(writer, log)
 	}
+	return nil
 }
 
-func renderTaskLog(db *sql.DB, start, end time.Time, limit int, plain bool) (string, error) {
-	entries, err := fetchTLEntriesBetweenTSFromDB(db, start, end, limit)
+func getTaskLog(db *sql.DB, start, end time.Time, limit int, plain bool) (string, error) {
+	entries, err := pers.FetchTLEntriesBetweenTS(db, start, end, limit)
 	if err != nil {
 		return "", err
 	}
 
 	var numEntriesInTable int
 
 	if len(entries) == 0 {
 		numEntriesInTable = 1
 	} else {
 		numEntriesInTable = len(entries)
 	}
 
 	data := make([][]string, numEntriesInTable)
 
 	if len(entries) == 0 {
 		data[0] = []string{
-			RightPadTrim("", 20, false),
-			RightPadTrim("", 40, false),
-			RightPadTrim("", 39, false),
-			RightPadTrim("", logTimeCharsBudget, false),
+			utils.RightPadTrim("", 20, false),
+			utils.RightPadTrim("", 40, false),
+			utils.RightPadTrim("", 39, false),
+			utils.RightPadTrim("", logTimeCharsBudget, false),
 		}
 	}
 
 	var timeSpentStr string
 
 	rs := getReportStyles(plain)
 	styleCache := make(map[string]lipgloss.Style)
 
 	for i, entry := range entries {
-		timeSpentStr = humanizeDuration(entry.secsSpent)
+		timeSpentStr = types.HumanizeDuration(entry.SecsSpent)
 
 		if plain {
 			data[i] = []string{
-				RightPadTrim(entry.taskSummary, 20, false),
-				RightPadTrim(entry.comment, 40, false),
-				fmt.Sprintf("%s  ...  %s", entry.beginTs.Format(timeFormat), entry.endTs.Format(timeFormat)),
-				RightPadTrim(timeSpentStr, logTimeCharsBudget, false),
+				utils.RightPadTrim(entry.TaskSummary, 20, false),
+				utils.RightPadTrim(entry.Comment, 40, false),
+				fmt.Sprintf("%s  ...  %s", entry.BeginTS.Format(timeFormat), entry.EndTS.Format(timeFormat)),
+				utils.RightPadTrim(timeSpentStr, logTimeCharsBudget, false),
 			}
 		} else {
-			rowStyle, ok := styleCache[entry.taskSummary]
+			rowStyle, ok := styleCache[entry.TaskSummary]
 			if !ok {
-				rowStyle = getDynamicStyle(entry.taskSummary)
-				styleCache[entry.taskSummary] = rowStyle
+				rowStyle = getDynamicStyle(entry.TaskSummary)
+				styleCache[entry.TaskSummary] = rowStyle
 			}
 			data[i] = []string{
-				rowStyle.Render(RightPadTrim(entry.taskSummary, 20, false)),
-				rowStyle.Render(RightPadTrim(entry.comment, 40, false)),
-				rowStyle.Render(fmt.Sprintf("%s  ...  %s", entry.beginTs.Format(timeFormat), entry.endTs.Format(timeFormat))),
-				rowStyle.Render(RightPadTrim(timeSpentStr, logTimeCharsBudget, false)),
+				rowStyle.Render(utils.RightPadTrim(entry.TaskSummary, 20, false)),
+				rowStyle.Render(utils.RightPadTrim(entry.Comment, 40, false)),
+				rowStyle.Render(fmt.Sprintf("%s  ...  %s", entry.BeginTS.Format(timeFormat), entry.EndTS.Format(timeFormat))),
+				rowStyle.Render(utils.RightPadTrim(timeSpentStr, logTimeCharsBudget, false)),
 			}
 		}
 	}
 
 	b := bytes.Buffer{}
 	table := tablewriter.NewWriter(&b)
 
 	headerValues := []string{"Task", "Comment", "Duration", "TimeSpent"}
 	headers := make([]string, len(headerValues))
 	for i, h := range headerValues {
diff --git a/internal/ui/model.go b/internal/ui/model.go
index 2279da0..0516e15 100644
--- a/internal/ui/model.go
+++ b/internal/ui/model.go
@@ -1,20 +1,21 @@
 package ui
 
 import (
 	"database/sql"
 	"time"
 
 	"github.com/charmbracelet/bubbles/list"
 	"github.com/charmbracelet/bubbles/textinput"
 	"github.com/charmbracelet/bubbles/viewport"
 	tea "github.com/charmbracelet/bubbletea"
+	"github.com/dhth/hours/internal/types"
 )
 
 type trackingStatus uint
 
 const (
 	trackingInactive trackingStatus = iota
 	trackingActive
 )
 
 type dBChange uint
@@ -75,51 +76,51 @@ const (
 )
 
 const (
 	timeFormat         = "2006/01/02 15:04"
 	timeOnlyFormat     = "15:04"
 	dayFormat          = "Monday"
 	friendlyTimeFormat = "Mon, 15:04"
 	dateFormat         = "2006/01/02"
 )
 
-type model struct {
+type Model struct {
 	activeView             stateView
 	lastView               stateView
 	db                     *sql.DB
 	activeTasksList        list.Model
 	inactiveTasksList      list.Model
-	activeTaskMap          map[int]*task
+	activeTaskMap          map[int]*types.Task
 	activeTaskIndexMap     map[int]int
 	activeTLBeginTS        time.Time
 	activeTLEndTS          time.Time
 	tasksFetched           bool
 	taskLogList            list.Model
 	trackingInputs         []textinput.Model
 	trackingFocussedField  timeTrackingFormField
 	taskInputs             []textinput.Model
 	taskMgmtContext        taskMgmtContext
 	taskInputFocussedField taskInputField
 	helpVP                 viewport.Model
 	helpVPReady            bool
 	lastChange             dBChange
 	changesLocked          bool
-	activeTaskId           int
+	activeTaskID           int
 	tasklogSaveType        tasklogSaveType
 	message                string
 	messages               []string
 	showHelpIndicator      bool
 	terminalHeight         int
 	trackingActive         bool
 }
 
-func (m model) Init() tea.Cmd {
+func (m Model) Init() tea.Cmd {
 	return tea.Batch(
 		hideHelp(time.Minute*1),
 		fetchTasks(m.db, true),
 		fetchTaskLogEntries(m.db),
 		fetchTasks(m.db, false),
 	)
 }
 
 type recordsModel struct {
 	db       *sql.DB
@@ -128,13 +129,13 @@ type recordsModel struct {
 	end      time.Time
 	period   string
 	numDays  int
 	plain    bool
 	report   string
 	quitting bool
 	busy     bool
 	err      error
 }
 
-func (m recordsModel) Init() tea.Cmd {
+func (recordsModel) Init() tea.Cmd {
 	return nil
 }
diff --git a/internal/ui/msgs.go b/internal/ui/msgs.go
index 153d6d7..8d82e95 100644
--- a/internal/ui/msgs.go
+++ b/internal/ui/msgs.go
@@ -1,77 +1,81 @@
 package ui
 
-import "time"
+import (
+	"time"
+
+	"github.com/dhth/hours/internal/types"
+)
 
 type HideHelpMsg struct{}
 
 type trackingToggledMsg struct {
-	taskId    int
+	taskID    int
 	finished  bool
 	secsSpent int
 	err       error
 }
 
 type taskRepUpdatedMsg struct {
-	tsk *task
+	tsk *types.Task
 	err error
 }
 
 type manualTaskLogInserted struct {
-	taskId int
+	taskID int
 	err    error
 }
 
 type tlBeginTSUpdatedMsg struct {
 	beginTS time.Time
 	err     error
 }
 
 type activeTaskLogDeletedMsg struct {
 	err error
 }
 
 type activeTaskFetchedMsg struct {
-	activeTaskId int
+	activeTaskID int
 	beginTs      time.Time
 	noneActive   bool
 	err          error
 }
 
 type taskLogEntriesFetchedMsg struct {
-	entries []taskLogEntry
+	entries []types.TaskLogEntry
 	err     error
 }
 
 type taskCreatedMsg struct {
 	err error
 }
 
 type taskUpdatedMsg struct {
-	tsk     *task
+	tsk     *types.Task
 	summary string
 	err     error
 }
 
 type taskActiveStatusUpdated struct {
-	tsk    *task
+	tsk    *types.Task
 	active bool
 	err    error
 }
 
 type taskLogEntryDeletedMsg struct {
-	entry *taskLogEntry
+	entry *types.TaskLogEntry
 	err   error
 }
 
 type tasksFetched struct {
-	tasks  []task
+	tasks  []types.Task
 	active bool
 	err    error
 }
 
 type recordsDataFetchedMsg struct {
 	start  time.Time
 	end    time.Time
 	report string
 	err    error
 }
diff --git a/internal/ui/queries.go b/internal/ui/queries.go
deleted file mode 100644
index 49d5909..0000000
--- a/internal/ui/queries.go
+++ /dev/null
@@ -1,522 +0,0 @@
-package ui
-
-import (
-	"database/sql"
-	"errors"
-	"fmt"
-	"time"
-)
-
-func insertNewTLInDB(db *sql.DB, taskId int, beginTs time.Time) error {
-	stmt, err := db.Prepare(`
-INSERT INTO task_log (task_id, begin_ts, active)
-VALUES (?, ?, ?);
-`)
-	if err != nil {
-		return err
-	}
-	defer stmt.Close()
-
-	_, err = stmt.Exec(taskId, beginTs.UTC(), true)
-	if err != nil {
-		return err
-	}
-
-	return nil
-}
-
-func updateTLBeginTSInDB(db *sql.DB, beginTs time.Time) error {
-	stmt, err := db.Prepare(`
-UPDATE task_log SET begin_ts=?
-WHERE active is true;
-`)
-	if err != nil {
-		return err
-	}
-	defer stmt.Close()
-
-	_, err = stmt.Exec(beginTs.UTC(), true)
-	if err != nil {
-		return err
-	}
-
-	return nil
-}
-
-func deleteActiveTLInDB(db *sql.DB) error {
-	stmt, err := db.Prepare(`
-DELETE FROM task_log
-WHERE active=true;
-`)
-	if err != nil {
-		return err
-	}
-	defer stmt.Close()
-
-	_, err = stmt.Exec()
-
-	return err
-}
-
-func updateActiveTLInDB(db *sql.DB, taskLogId int, taskId int, beginTs, endTs time.Time, secsSpent int, comment string) error {
-	tx, err := db.Begin()
-	if err != nil {
-		return err
-	}
-	defer func() {
-		_ = tx.Rollback()
-	}()
-
-	stmt, err := tx.Prepare(`
-UPDATE task_log
-SET active = 0,
-    begin_ts = ?,
-    end_ts = ?,
-    secs_spent = ?,
-    comment = ?
-WHERE id = ?
-AND active = 1;
-`)
-	if err != nil {
-		return err
-	}
-	defer stmt.Close()
-
-	_, err = stmt.Exec(beginTs.UTC(), endTs.UTC(), secsSpent, comment, taskLogId)
-	if err != nil {
-		return err
-	}
-
-	tStmt, err := tx.Prepare(`
-UPDATE task
-SET secs_spent = secs_spent+?,
-    updated_at = ?
-WHERE id = ?;
-    `)
-	if err != nil {
-		return err
-	}
-	defer tStmt.Close()
-
-	_, err = tStmt.Exec(secsSpent, time.Now().UTC(), taskId)
-	if err != nil {
-		return err
-	}
-
-	err = tx.Commit()
-	if err != nil {
-		return err
-	}
-
-	return nil
-}
-
-func insertManualTLInDB(db *sql.DB, taskId int, beginTs time.Time, endTs time.Time, comment string) error {
-	secsSpent := int(endTs.Sub(beginTs).Seconds())
-	tx, err := db.Begin()
-	if err != nil {
-		return err
-	}
-	defer func() {
-		_ = tx.Rollback()
-	}()
-
-	stmt, err := tx.Prepare(`
-INSERT INTO task_log (task_id, begin_ts, end_ts, secs_spent, comment, active)
-VALUES (?, ?, ?, ?, ?, ?);
-`)
-	if err != nil {
-		return err
-	}
-	defer stmt.Close()
-
-	_, err = stmt.Exec(taskId, beginTs.UTC(), endTs.UTC(), secsSpent, comment, false)
-	if err != nil {
-		return err
-	}
-
-	tStmt, err := tx.Prepare(`
-UPDATE task
-SET secs_spent = secs_spent+?,
-    updated_at = ?
-WHERE id = ?;
-    `)
-	if err != nil {
-		return err
-	}
-	defer tStmt.Close()
-
-	_, err = tStmt.Exec(secsSpent, time.Now().UTC(), taskId)
-	if err != nil {
-		return err
-	}
-
-	err = tx.Commit()
-	if err != nil {
-		return err
-	}
-
-	return nil
-}
-
-func fetchActiveTaskFromDB(db *sql.DB) (activeTaskDetails, error) {
-	row := db.QueryRow(`
-SELECT t.id, t.summary, tl.begin_ts
-FROM task_log tl left join task t on tl.task_id = t.id
-WHERE tl.active=true;
-`)
-
-	var activeTaskDetails activeTaskDetails
-	err := row.Scan(
-		&activeTaskDetails.taskId,
-		&activeTaskDetails.taskSummary,
-		&activeTaskDetails.lastLogEntryBeginTs,
-	)
-	if errors.Is(err, sql.ErrNoRows) {
-		activeTaskDetails.taskId = -1
-		return activeTaskDetails, nil
-	} else if err != nil {
-		return activeTaskDetails, err
-	}
-	activeTaskDetails.lastLogEntryBeginTs = activeTaskDetails.lastLogEntryBeginTs.Local()
-	return activeTaskDetails, nil
-}
-
-func insertTaskInDB(db *sql.DB, summary string) error {
-	stmt, err := db.Prepare(`
-INSERT into task (summary, active, created_at, updated_at)
-VALUES (?, true, ?, ?);
-`)
-	if err != nil {
-		return err
-	}
-	defer stmt.Close()
-
-	now := time.Now().UTC()
-	_, err = stmt.Exec(summary, now, now)
-	if err != nil {
-		return err
-	}
-	return nil
-}
-
-func updateTaskInDB(db *sql.DB, id int, summary string) error {
-	stmt, err := db.Prepare(`
-UPDATE task
-SET summary = ?,
-    updated_at = ?
-WHERE id = ?
-`)
-	if err != nil {
-		return err
-	}
-	defer stmt.Close()
-
-	_, err = stmt.Exec(summary, time.Now().UTC(), id)
-	if err != nil {
-		return err
-	}
-	return nil
-}
-
-func updateTaskActiveStatusInDB(db *sql.DB, id int, active bool) error {
-	stmt, err := db.Prepare(`
-UPDATE task
-SET active = ?,
-    updated_at = ?
-WHERE id = ?
-`)
-	if err != nil {
-		return err
-	}
-	defer stmt.Close()
-
-	_, err = stmt.Exec(active, time.Now().UTC(), id)
-	if err != nil {
-		return err
-	}
-	return nil
-}
-
-func updateTaskDataFromDB(db *sql.DB, t *task) error {
-	row := db.QueryRow(`
-SELECT secs_spent, updated_at
-FROM task
-WHERE id=?;
-    `, t.id)
-
-	err := row.Scan(
-		&t.secsSpent,
-		&t.updatedAt,
-	)
-	if err != nil {
-		return err
-	}
-	return nil
-}
-
-func fetchTasksFromDB(db *sql.DB, active bool, limit int) ([]task, error) {
-	var tasks []task
-
-	rows, err := db.Query(`
-SELECT id, summary, secs_spent, created_at, updated_at, active
-FROM task
-WHERE active=?
-ORDER by updated_at DESC
-LIMIT ?;
-    `, active, limit)
-	if err != nil {
-		return nil, err
-	}
-	defer rows.Close()
-
-	for rows.Next() {
-		var entry task
-		err = rows.Scan(&entry.id,
-			&entry.summary,
-			&entry.secsSpent,
-			&entry.createdAt,
-			&entry.updatedAt,
-			&entry.active,
-		)
-		if err != nil {
-			return nil, err
-		}
-		entry.createdAt = entry.createdAt.Local()
-		entry.updatedAt = entry.updatedAt.Local()
-		tasks = append(tasks, entry)
-
-	}
-	return tasks, nil
-}
-
-func fetchTLEntriesFromDB(db *sql.DB, desc bool, limit int) ([]taskLogEntry, error) {
-	var logEntries []taskLogEntry
-
-	var order string
-	if desc {
-		order = "DESC"
-	} else {
-		order = "ASC"
-	}
-	query := fmt.Sprintf(`
-SELECT tl.id, tl.task_id, t.summary, tl.begin_ts, tl.end_ts, tl.secs_spent, tl.comment
-FROM task_log tl left join task t on tl.task_id=t.id
-WHERE tl.active=false
-ORDER by tl.begin_ts %s
-LIMIT ?;
-`, order)
-
-	rows, err := db.Query(query, limit)
-	if err != nil {
-		return nil, err
-	}
-	defer rows.Close()
-
-	for rows.Next() {
-		var entry taskLogEntry
-		err = rows.Scan(&entry.id,
-			&entry.taskId,
-			&entry.taskSummary,
-			&entry.beginTs,
-			&entry.endTs,
-			&entry.secsSpent,
-			&entry.comment,
-		)
-		if err != nil {
-			return nil, err
-		}
-		entry.beginTs = entry.beginTs.Local()
-		entry.endTs = entry.endTs.Local()
-		logEntries = append(logEntries, entry)
-
-	}
-	return logEntries, nil
-}
-
-func fetchTLEntriesBetweenTSFromDB(db *sql.DB, beginTs, endTs time.Time, limit int) ([]taskLogEntry, error) {
-	var logEntries []taskLogEntry
-
-	rows, err := db.Query(`
-SELECT tl.id, tl.task_id, t.summary, tl.begin_ts, tl.end_ts, tl.secs_spent, tl.comment
-FROM task_log tl left join task t on tl.task_id=t.id
-WHERE tl.active=false
-AND tl.end_ts >= ?
-AND tl.end_ts < ?
-ORDER by tl.begin_ts ASC LIMIT ?;
-    `, beginTs.UTC(), endTs.UTC(), limit)
-	if err != nil {
-		return nil, err
-	}
-	defer rows.Close()
-
-	for rows.Next() {
-		var entry taskLogEntry
-		err = rows.Scan(&entry.id,
-			&entry.taskId,
-			&entry.taskSummary,
-			&entry.beginTs,
-			&entry.endTs,
-			&entry.secsSpent,
-			&entry.comment,
-		)
-		if err != nil {
-			return nil, err
-		}
-		entry.beginTs = entry.beginTs.Local()
-		entry.endTs = entry.endTs.Local()
-		logEntries = append(logEntries, entry)
-
-	}
-	return logEntries, nil
-}
-
-func fetchStatsFromDB(db *sql.DB, limit int) ([]taskReportEntry, error) {
-	rows, err := db.Query(`
-SELECT tl.task_id, t.summary, COUNT(tl.id) as num_entries, t.secs_spent
-from task_log tl
-LEFT JOIN task t on tl.task_id = t.id
-GROUP BY tl.task_id
-ORDER BY t.secs_spent DESC
-limit ?;
-`, limit)
-	if err != nil {
-		return nil, err
-	}
-	defer rows.Close()
-
-	var tLE []taskReportEntry
-
-	for rows.Next() {
-		var entry taskReportEntry
-		err = rows.Scan(
-			&entry.taskId,
-			&entry.taskSummary,
-			&entry.numEntries,
-			&entry.secsSpent,
-		)
-		if err != nil {
-			return nil, err
-		}
-		tLE = append(tLE, entry)
-
-	}
-	return tLE, nil
-}
-
-func fetchStatsBetweenTSFromDB(db *sql.DB, beginTs, endTs time.Time, limit int) ([]taskReportEntry, error) {
-	rows, err := db.Query(`
-SELECT tl.task_id, t.summary, COUNT(tl.id) as num_entries,  SUM(tl.secs_spent) AS secs_spent
-FROM task_log tl 
-LEFT JOIN task t ON tl.task_id = t.id
-WHERE tl.end_ts >= ? AND tl.end_ts < ?
-GROUP BY tl.task_id
-ORDER BY secs_spent DESC
-LIMIT ?;
-`, beginTs.UTC(), endTs.UTC(), limit)
-	if err != nil {
-		return nil, err
-	}
-	defer rows.Close()
-
-	var tLE []taskReportEntry
-
-	for rows.Next() {
-		var entry taskReportEntry
-		err = rows.Scan(
-			&entry.taskId,
-			&entry.taskSummary,
-			&entry.numEntries,
-			&entry.secsSpent,
-		)
-		if err != nil {
-			return nil, err
-		}
-		tLE = append(tLE, entry)
-
-	}
-	return tLE, nil
-}
-
-func fetchReportBetweenTSFromDB(db *sql.DB, beginTs, endTs time.Time, limit int) ([]taskReportEntry, error) {
-	rows, err := db.Query(`
-SELECT tl.task_id, t.summary, COUNT(tl.id) as num_entries,  SUM(tl.secs_spent) AS secs_spent
-FROM task_log tl 
-LEFT JOIN task t ON tl.task_id = t.id
-WHERE tl.end_ts >= ? AND tl.end_ts < ?
-GROUP BY tl.task_id
-ORDER BY t.updated_at ASC
-LIMIT ?;
-`, beginTs.UTC(), endTs.UTC(), limit)
-	if err != nil {
-		return nil, err
-	}
-	defer rows.Close()
-
-	var tLE []taskReportEntry
-
-	for rows.Next() {
-		var entry taskReportEntry
-		err = rows.Scan(
-			&entry.taskId,
-			&entry.taskSummary,
-			&entry.numEntries,
-			&entry.secsSpent,
-		)
-		if err != nil {
-			return nil, err
-		}
-		tLE = append(tLE, entry)
-
-	}
-	return tLE, nil
-}
-
-func deleteEntry(db *sql.DB, entry *taskLogEntry) error {
-	secsSpent := entry.secsSpent
-
-	tx, err := db.Begin()
-	if err != nil {
-		return err
-	}
-	defer func() {
-		_ = tx.Rollback()
-	}()
-
-	stmt, err := tx.Prepare(`
-DELETE from task_log
-WHERE ID=?;
-`)
-	if err != nil {
-		return err
-	}
-	defer stmt.Close()
-
-	_, err = stmt.Exec(entry.id)
-	if err != nil {
-		return err
-	}
-
-	tStmt, err := tx.Prepare(`
-UPDATE task
-SET secs_spent = secs_spent-?,
-    updated_at = ?
-WHERE id = ?;
-    `)
-	if err != nil {
-		return err
-	}
-	defer tStmt.Close()
-
-	_, err = tStmt.Exec(secsSpent, time.Now().UTC(), entry.taskId)
-	if err != nil {
-		return err
-	}
-
-	err = tx.Commit()
-	if err != nil {
-		return err
-	}
-
-	return nil
-}
diff --git a/internal/ui/render_helpers.go b/internal/ui/render_helpers.go
deleted file mode 100644
index b8cbfef..0000000
--- a/internal/ui/render_helpers.go
+++ /dev/null
@@ -1,60 +0,0 @@
-package ui
-
-import (
-	"fmt"
-	"time"
-
-	"github.com/dustin/go-humanize"
-)
-
-func (t *task) updateTitle() {
-	var trackingIndicator string
-	if t.trackingActive {
-		trackingIndicator = "⏲ "
-	}
-
-	t.title = trackingIndicator + t.summary
-}
-
-func (t *task) updateDesc() {
-	var timeSpent string
-
-	if t.secsSpent != 0 {
-		timeSpent = "worked on for " + humanizeDuration(t.secsSpent)
-	} else {
-		timeSpent = "no time spent"
-	}
-	lastUpdated := fmt.Sprintf("last updated: %s", humanize.Time(t.updatedAt))
-
-	t.desc = fmt.Sprintf("%s %s", RightPadTrim(lastUpdated, 60, true), timeSpent)
-}
-
-func (tl *taskLogEntry) updateTitle() {
-	tl.title = Trim(tl.comment, 60)
-}
-
-func (tl *taskLogEntry) updateDesc() {
-	timeSpentStr := humanizeDuration(tl.secsSpent)
-
-	var timeStr string
-	var durationMsg string
-
-	endTSRelative := getTSRelative(tl.endTs, time.Now())
-
-	switch endTSRelative {
-	case tsFromToday:
-		durationMsg = fmt.Sprintf("%s  ...  %s", tl.beginTs.Format(timeOnlyFormat), tl.endTs.Format(timeOnlyFormat))
-	case tsFromYesterday:
-		durationMsg = "Yesterday"
-	case tsFromThisWeek:
-		durationMsg = tl.endTs.Format(dayFormat)
-	default:
-		durationMsg = humanize.Time(tl.endTs)
-	}
-
-	timeStr = fmt.Sprintf("%s (%s)",
-		RightPadTrim(durationMsg, 40, true),
-		timeSpentStr)
-
-	tl.desc = fmt.Sprintf("%s %s", RightPadTrim("["+tl.taskSummary+"]", 60, true), timeStr)
-}
diff --git a/internal/ui/report.go b/internal/ui/report.go
index 8e4ee1f..f4ee2dc 100644
--- a/internal/ui/report.go
+++ b/internal/ui/report.go
@@ -1,80 +1,85 @@
 package ui
 
 import (
 	"bytes"
 	"database/sql"
+	"errors"
 	"fmt"
 	"io"
-	"os"
 	"time"
 
 	tea "github.com/charmbracelet/bubbletea"
 	"github.com/charmbracelet/lipgloss"
+	pers "github.com/dhth/hours/internal/persistence"
+	"github.com/dhth/hours/internal/types"
+	"github.com/dhth/hours/internal/utils"
 	"github.com/olekukonko/tablewriter"
 )
 
+var errCouldntGenerateReport = errors.New("couldn't generate report")
+
 const (
 	reportTimeCharsBudget = 6
 )
 
-func RenderReport(db *sql.DB, writer io.Writer, plain bool, period string, agg bool, interactive bool) {
+func RenderReport(db *sql.DB, writer io.Writer, plain bool, period string, agg bool, interactive bool) error {
 	if period == "" {
-		return
+		return nil
 	}
 
 	var fullWeek bool
 	if interactive {
 		fullWeek = true
 	}
-	ts, err := getTimePeriod(period, time.Now(), fullWeek)
+	ts, err := types.GetTimePeriod(period, time.Now(), fullWeek)
 	if err != nil {
-		fmt.Printf("error: %s\n", err)
-		os.Exit(1)
+		return err
 	}
 
 	var report string
 	var analyticsType recordsType
 
 	if agg {
 		analyticsType = reportAggRecords
-		report, err = getReportAgg(db, ts.start, ts.numDays, plain)
+		report, err = getReportAgg(db, ts.Start, ts.NumDays, plain)
 	} else {
 		analyticsType = reportRecords
-		report, err = getReport(db, ts.start, ts.numDays, plain)
+		report, err = getReport(db, ts.Start, ts.NumDays, plain)
 	}
 	if err != nil {
-		fmt.Printf("Something went wrong generating the report: %s\n", err)
+		return fmt.Errorf("%w: %s", errCouldntGenerateReport, err.Error())
 	}
 
 	if interactive {
-		p := tea.NewProgram(initialRecordsModel(analyticsType, db, ts.start, ts.end, plain, period, ts.numDays, report))
-		if _, err := p.Run(); err != nil {
-			fmt.Printf("Alas, there has been an error: %v", err)
-			os.Exit(1)
+		p := tea.NewProgram(initialRecordsModel(analyticsType, db, ts.Start, ts.End, plain, period, ts.NumDays, report))
+		_, err := p.Run()
+		if err != nil {
+			return err
 		}
 	} else {
 		fmt.Fprint(writer, report)
 	}
+	return nil
 }
 
 func getReport(db *sql.DB, start time.Time, numDays int, plain bool) (string, error) {
 	day := start
 	var nextDay time.Time
 
 	var maxEntryForADay int
-	reportData := make(map[int][]taskLogEntry)
+	reportData := make(map[int][]types.TaskLogEntry)
 
 	noEntriesFound := true
 	for i := 0; i < numDays; i++ {
 		nextDay = day.AddDate(0, 0, 1)
-		taskLogEntries, err := fetchTLEntriesBetweenTSFromDB(db, day, nextDay, 100)
+		taskLogEntries, err := pers.FetchTLEntriesBetweenTS(db, day, nextDay, 100)
 		if err != nil {
 			return "", err
 		}
 		if noEntriesFound && len(taskLogEntries) > 0 {
 			noEntriesFound = false
 		}
 
 		day = nextDay
 		reportData[i] = taskLogEntries
 
@@ -107,57 +112,57 @@ func getReport(db *sql.DB, start time.Time, numDays int, plain bool) (string, er
 	default:
 		summaryBudget = 16
 	}
 
 	styleCache := make(map[string]lipgloss.Style)
 	for rowIndex := 0; rowIndex < maxEntryForADay; rowIndex++ {
 		row := make([]string, numDays)
 		for colIndex := 0; colIndex < numDays; colIndex++ {
 			if rowIndex >= len(reportData[colIndex]) {
 				row[colIndex] = fmt.Sprintf("%s  %s",
-					RightPadTrim("", summaryBudget, false),
-					RightPadTrim("", reportTimeCharsBudget, false),
+					utils.RightPadTrim("", summaryBudget, false),
+					utils.RightPadTrim("", reportTimeCharsBudget, false),
 				)
 				continue
 			}
 
 			tr := reportData[colIndex][rowIndex]
-			timeSpentStr := humanizeDuration(tr.secsSpent)
+			timeSpentStr := types.HumanizeDuration(tr.SecsSpent)
 
 			if plain {
 				row[colIndex] = fmt.Sprintf("%s  %s",
-					RightPadTrim(tr.taskSummary, summaryBudget, false),
-					RightPadTrim(timeSpentStr, reportTimeCharsBudget, false),
+					utils.RightPadTrim(tr.TaskSummary, summaryBudget, false),
+					utils.RightPadTrim(timeSpentStr, reportTimeCharsBudget, false),
 				)
 			} else {
-				rowStyle, ok := styleCache[tr.taskSummary]
+				rowStyle, ok := styleCache[tr.TaskSummary]
 
 				if !ok {
-					rowStyle = getDynamicStyle(tr.taskSummary)
-					styleCache[tr.taskSummary] = rowStyle
+					rowStyle = getDynamicStyle(tr.TaskSummary)
+					styleCache[tr.TaskSummary] = rowStyle
 				}
 
 				row[colIndex] = fmt.Sprintf("%s  %s",
-					rowStyle.Render(RightPadTrim(tr.taskSummary, summaryBudget, false)),
-					rowStyle.Render(RightPadTrim(timeSpentStr, reportTimeCharsBudget, false)),
+					rowStyle.Render(utils.RightPadTrim(tr.TaskSummary, summaryBudget, false)),
+					rowStyle.Render(utils.RightPadTrim(timeSpentStr, reportTimeCharsBudget, false)),
 				)
 			}
-			totalSecsPerDay[colIndex] += tr.secsSpent
+			totalSecsPerDay[colIndex] += tr.SecsSpent
 		}
 		data[rowIndex] = row
 	}
 
 	totalTimePerDay := make([]string, numDays)
 
 	for i, ts := range totalSecsPerDay {
 		if ts != 0 {
-			totalTimePerDay[i] = rs.footerStyle.Render(humanizeDuration(ts))
+			totalTimePerDay[i] = rs.footerStyle.Render(types.HumanizeDuration(ts))
 		} else {
 			totalTimePerDay[i] = " "
 		}
 	}
 
 	b := bytes.Buffer{}
 	table := tablewriter.NewWriter(&b)
 
 	headersValues := make([]string, numDays)
 
@@ -188,26 +193,26 @@ func getReport(db *sql.DB, start time.Time, numDays int, plain bool) (string, er
 	table.Render()
 
 	return b.String(), nil
 }
 
 func getReportAgg(db *sql.DB, start time.Time, numDays int, plain bool) (string, error) {
 	day := start
 	var nextDay time.Time
 
 	var maxEntryForADay int
-	reportData := make(map[int][]taskReportEntry)
+	reportData := make(map[int][]types.TaskReportEntry)
 
 	noEntriesFound := true
 	for i := 0; i < numDays; i++ {
 		nextDay = day.AddDate(0, 0, 1)
-		taskLogEntries, err := fetchReportBetweenTSFromDB(db, day, nextDay, 100)
+		taskLogEntries, err := pers.FetchReportBetweenTS(db, day, nextDay, 100)
 		if err != nil {
 			return "", err
 		}
 		if noEntriesFound && len(taskLogEntries) > 0 {
 			noEntriesFound = false
 		}
 
 		day = nextDay
 		reportData[i] = taskLogEntries
 		if len(taskLogEntries) > maxEntryForADay {
@@ -239,54 +244,54 @@ func getReportAgg(db *sql.DB, start time.Time, numDays int, plain bool) (string,
 	default:
 		summaryBudget = 16
 	}
 
 	styleCache := make(map[string]lipgloss.Style)
 	for rowIndex := 0; rowIndex < maxEntryForADay; rowIndex++ {
 		row := make([]string, numDays)
 		for colIndex := 0; colIndex < numDays; colIndex++ {
 			if rowIndex >= len(reportData[colIndex]) {
 				row[colIndex] = fmt.Sprintf("%s  %s",
-					RightPadTrim("", summaryBudget, false),
-					RightPadTrim("", reportTimeCharsBudget, false),
+					utils.RightPadTrim("", summaryBudget, false),
+					utils.RightPadTrim("", reportTimeCharsBudget, false),
 				)
 				continue
 			}
 
 			tr := reportData[colIndex][rowIndex]
-			timeSpentStr := humanizeDuration(tr.secsSpent)
+			timeSpentStr := types.HumanizeDuration(tr.SecsSpent)
 
 			if plain {
 				row[colIndex] = fmt.Sprintf("%s  %s",
-					RightPadTrim(tr.taskSummary, summaryBudget, false),
-					RightPadTrim(timeSpentStr, reportTimeCharsBudget, false),
+					utils.RightPadTrim(tr.TaskSummary, summaryBudget, false),
+					utils.RightPadTrim(timeSpentStr, reportTimeCharsBudget, false),
 				)
 			} else {
-				rowStyle, ok := styleCache[tr.taskSummary]
+				rowStyle, ok := styleCache[tr.TaskSummary]
 				if !ok {
-					rowStyle = getDynamicStyle(tr.taskSummary)
-					styleCache[tr.taskSummary] = rowStyle
+					rowStyle = getDynamicStyle(tr.TaskSummary)
+					styleCache[tr.TaskSummary] = rowStyle
 				}
 
 				row[colIndex] = fmt.Sprintf("%s  %s",
-					rowStyle.Render(RightPadTrim(tr.taskSummary, summaryBudget, false)),
-					rowStyle.Render(RightPadTrim(timeSpentStr, reportTimeCharsBudget, false)),
+					rowStyle.Render(utils.RightPadTrim(tr.TaskSummary, summaryBudget, false)),
+					rowStyle.Render(utils.RightPadTrim(timeSpentStr, reportTimeCharsBudget, false)),
 				)
 			}
-			totalSecsPerDay[colIndex] += tr.secsSpent
+			totalSecsPerDay[colIndex] += tr.SecsSpent
 		}
 		data[rowIndex] = row
 	}
 	totalTimePerDay := make([]string, numDays)
 	for i, ts := range totalSecsPerDay {
 		if ts != 0 {
-			totalTimePerDay[i] = rs.footerStyle.Render(humanizeDuration(ts))
+			totalTimePerDay[i] = rs.footerStyle.Render(types.HumanizeDuration(ts))
 		} else {
 			totalTimePerDay[i] = " "
 		}
 	}
 
 	b := bytes.Buffer{}
 	table := tablewriter.NewWriter(&b)
 
 	headersValues := make([]string, numDays)
 
diff --git a/internal/ui/stats.go b/internal/ui/stats.go
index 2997fe7..1439d56 100644
--- a/internal/ui/stats.go
+++ b/internal/ui/stats.go
@@ -1,136 +1,139 @@
 package ui
 
 import (
 	"bytes"
 	"database/sql"
+	"errors"
 	"fmt"
 	"io"
-	"os"
 	"time"
 
 	tea "github.com/charmbracelet/bubbletea"
 	"github.com/charmbracelet/lipgloss"
+	pers "github.com/dhth/hours/internal/persistence"
+	"github.com/dhth/hours/internal/types"
+	"github.com/dhth/hours/internal/utils"
 	"github.com/olekukonko/tablewriter"
 )
 
+var errCouldntGenerateStats = errors.New("couldn't generate stats")
+
 const (
 	statsLogEntriesLimit   = 10000
 	statsNumDaysUpperBound = 3650
 	statsTimeCharsBudget   = 6
+	periodAll              = "all"
 )
 
-func RenderStats(db *sql.DB, writer io.Writer, plain bool, period string, interactive bool) {
+func RenderStats(db *sql.DB, writer io.Writer, plain bool, period string, interactive bool) error {
 	if period == "" {
-		return
+		return nil
 	}
 
 	var stats string
 	var err error
 
-	if interactive && period == "all" {
-		fmt.Print("Interactive mode cannot be used when period='all'\n")
-		os.Exit(1)
+	if interactive && period == periodAll {
+		return fmt.Errorf("%w when period=all", errInteractiveModeNotApplicable)
 	}
 
-	if period == "all" {
+	if period == periodAll {
 		// TODO: find a better way for this, passing start, end for "all" doesn't make sense
-		stats, err = renderStats(db, period, time.Now(), time.Now(), plain)
+		stats, err = getStats(db, period, time.Now(), time.Now(), plain)
 		if err != nil {
-			fmt.Fprintf(writer, "Something went wrong generating the log: %s\n", err)
-			os.Exit(1)
+			return fmt.Errorf("%w: %s", errCouldntGenerateStats, err.Error())
 		}
 		fmt.Fprint(writer, stats)
-		return
+		return nil
 	}
 
 	var fullWeek bool
 	if interactive {
 		fullWeek = true
 	}
-	ts, tsErr := getTimePeriod(period, time.Now(), fullWeek)
-
-	if tsErr != nil {
-		fmt.Printf("error: %s\n", tsErr)
-		os.Exit(1)
-	}
-	stats, err = renderStats(db, period, ts.start, ts.end, plain)
+	ts, err := types.GetTimePeriod(period, time.Now(), fullWeek)
 	if err != nil {
-		fmt.Fprintf(writer, "Something went wrong generating the log: %s\n", err)
-		os.Exit(1)
+		return err
+	}
+
+	stats, err = getStats(db, period, ts.Start, ts.End, plain)
+	if err != nil {
+		return fmt.Errorf("%w: %s", errCouldntGenerateStats, err.Error())
 	}
 
 	if interactive {
-		p := tea.NewProgram(initialRecordsModel(reportStats, db, ts.start, ts.end, plain, period, ts.numDays, stats))
-		if _, err := p.Run(); err != nil {
-			fmt.Printf("Alas, there has been an error: %v", err)
-			os.Exit(1)
+		p := tea.NewProgram(initialRecordsModel(reportStats, db, ts.Start, ts.End, plain, period, ts.NumDays, stats))
+		_, err := p.Run()
+		if err != nil {
+			return err
 		}
 	} else {
 		fmt.Fprint(writer, stats)
 	}
+	return nil
 }
 
-func renderStats(db *sql.DB, period string, start, end time.Time, plain bool) (string, error) {
-	var entries []taskReportEntry
+func getStats(db *sql.DB, period string, start, end time.Time, plain bool) (string, error) {
+	var entries []types.TaskReportEntry
 	var err error
 
-	if period == "all" {
-		entries, err = fetchStatsFromDB(db, statsLogEntriesLimit)
+	if period == periodAll {
+		entries, err = pers.FetchStats(db, statsLogEntriesLimit)
 	} else {
-		entries, err = fetchStatsBetweenTSFromDB(db, start, end, statsLogEntriesLimit)
+		entries, err = pers.FetchStatsBetweenTS(db, start, end, statsLogEntriesLimit)
 	}
 
 	if err != nil {
 		return "", err
 	}
 
 	var numEntriesInTable int
 	if len(entries) == 0 {
 		numEntriesInTable = 1
 	} else {
 		numEntriesInTable = len(entries)
 	}
 
 	data := make([][]string, numEntriesInTable)
 	if len(entries) == 0 {
 		data[0] = []string{
-			RightPadTrim("", 20, false),
+			utils.RightPadTrim("", 20, false),
 			"",
-			RightPadTrim("", statsTimeCharsBudget, false),
+			utils.RightPadTrim("", statsTimeCharsBudget, false),
 		}
 	}
 
 	var timeSpentStr string
 
 	rs := getReportStyles(plain)
 	styleCache := make(map[string]lipgloss.Style)
 
 	for i, entry := range entries {
-		timeSpentStr = humanizeDuration(entry.secsSpent)
+		timeSpentStr = types.HumanizeDuration(entry.SecsSpent)
 
 		if plain {
 			data[i] = []string{
-				RightPadTrim(entry.taskSummary, 20, false),
-				fmt.Sprintf("%d", entry.numEntries),
-				RightPadTrim("", statsTimeCharsBudget, false),
+				utils.RightPadTrim(entry.TaskSummary, 20, false),
+				fmt.Sprintf("%d", entry.NumEntries),
+				utils.RightPadTrim(timeSpentStr, statsTimeCharsBudget, false),
 			}
 		} else {
-			rowStyle, ok := styleCache[entry.taskSummary]
+			rowStyle, ok := styleCache[entry.TaskSummary]
 			if !ok {
-				rowStyle = getDynamicStyle(entry.taskSummary)
-				styleCache[entry.taskSummary] = rowStyle
+				rowStyle = getDynamicStyle(entry.TaskSummary)
+				styleCache[entry.TaskSummary] = rowStyle
 			}
 			data[i] = []string{
-				rowStyle.Render(RightPadTrim(entry.taskSummary, 20, false)),
-				rowStyle.Render(fmt.Sprintf("%d", entry.numEntries)),
-				rowStyle.Render(RightPadTrim(timeSpentStr, statsTimeCharsBudget, false)),
+				rowStyle.Render(utils.RightPadTrim(entry.TaskSummary, 20, false)),
+				rowStyle.Render(fmt.Sprintf("%d", entry.NumEntries)),
+				rowStyle.Render(utils.RightPadTrim(timeSpentStr, statsTimeCharsBudget, false)),
 			}
 		}
 	}
 	b := bytes.Buffer{}
 	table := tablewriter.NewWriter(&b)
 
 	headerValues := []string{"Task", "#LogEntries", "TimeSpent"}
 	headers := make([]string, len(headerValues))
 	for i, h := range headerValues {
 		headers[i] = rs.headerStyle.Render(h)
diff --git a/internal/ui/styles.go b/internal/ui/styles.go
index 2094310..28de6e2 100644
--- a/internal/ui/styles.go
+++ b/internal/ui/styles.go
@@ -23,20 +23,21 @@ const (
 	recordsFooterColor       = "#ef8f62"
 	recordsBorderColor       = "#665c54"
 	initialHelpMsgColor      = "#a58390"
 	recordsDateRangeColor    = "#fabd2f"
 	recordsHelpColor         = "#928374"
 	helpMsgColor             = "#83a598"
 	helpViewTitleColor       = "#83a598"
 	helpHeaderColor          = "#83a598"
 	helpSectionColor         = "#bdae93"
 	warningColor             = "#fb4934"
+	fallbackTaskColor        = "#ada7ff"
 )
 
 var (
 	baseStyle = lipgloss.NewStyle().
 			PaddingLeft(1).
 			PaddingRight(1).
 			Foreground(lipgloss.Color(defaultBackgroundColor))
 
 	helpMsgStyle = lipgloss.NewStyle().
 			PaddingLeft(1).
@@ -130,21 +131,26 @@ var (
 		"#ffb4a2",
 		"#b8bb26",
 		"#ffc6ff",
 		"#4895ef",
 		"#83a598",
 		"#fabd2f",
 	}
 
 	getDynamicStyle = func(str string) lipgloss.Style {
 		h := fnv.New32()
-		h.Write([]byte(str))
+		_, err := h.Write([]byte(str))
+		if err != nil {
+			return lipgloss.NewStyle().
+				Foreground(lipgloss.Color(fallbackTaskColor))
+		}
+
 		hash := h.Sum32()
 
 		color := taskColors[hash%uint32(len(taskColors))]
 		return lipgloss.NewStyle().
 			Foreground(lipgloss.Color(color))
 	}
 
 	emptyStyle = lipgloss.NewStyle()
 
 	WarningStyle = lipgloss.NewStyle().
diff --git a/internal/ui/types.go b/internal/ui/types.go
deleted file mode 100644
index 8ac0ccd..0000000
--- a/internal/ui/types.go
+++ /dev/null
@@ -1,66 +0,0 @@
-package ui
-
-import (
-	"time"
-)
-
-type task struct {
-	id             int
-	summary        string
-	createdAt      time.Time
-	updatedAt      time.Time
-	trackingActive bool
-	secsSpent      int
-	active         bool
-	title          string
-	desc           string
-}
-
-type taskLogEntry struct {
-	id          int
-	taskId      int
-	taskSummary string
-	beginTs     time.Time
-	endTs       time.Time
-	secsSpent   int
-	comment     string
-	title       string
-	desc        string
-}
-
-type activeTaskDetails struct {
-	taskId              int
-	taskSummary         string
-	lastLogEntryBeginTs time.Time
-}
-
-type taskReportEntry struct {
-	taskId      int
-	taskSummary string
-	numEntries  int
-	secsSpent   int
-}
-
-func (t task) Title() string {
-	return t.title
-}
-
-func (t task) Description() string {
-	return t.desc
-}
-
-func (t task) FilterValue() string {
-	return t.summary
-}
-
-func (e taskLogEntry) Title() string {
-	return e.title
-}
-
-func (e taskLogEntry) Description() string {
-	return e.desc
-}
-
-func (e taskLogEntry) FilterValue() string {
-	return e.comment
-}
diff --git a/internal/ui/ui.go b/internal/ui/ui.go
index 4cac6d8..7171a24 100644
--- a/internal/ui/ui.go
+++ b/internal/ui/ui.go
@@ -1,26 +1,29 @@
 package ui
 
 import (
 	"database/sql"
+	"errors"
 	"fmt"
 	"os"
 
 	tea "github.com/charmbracelet/bubbletea"
 )
 
-func RenderUI(db *sql.DB) {
+var errFailedToConfigureDebugging = errors.New("failed to configure debugging")
+
+func RenderUI(db *sql.DB) error {
 	if len(os.Getenv("DEBUG")) > 0 {
 		f, err := tea.LogToFile("debug.log", "debug")
 		if err != nil {
-			fmt.Println("fatal:", err)
-			os.Exit(1)
+			return fmt.Errorf("%w: %s", errFailedToConfigureDebugging, err.Error())
 		}
 		defer f.Close()
 	}
 
 	p := tea.NewProgram(InitialModel(db), tea.WithAltScreen())
-	if _, err := p.Run(); err != nil {
-		fmt.Printf("Alas, there has been an error: %v", err)
-		os.Exit(1)
+	_, err := p.Run()
+	if err != nil {
+		return err
 	}
+	return nil
 }
diff --git a/internal/ui/update.go b/internal/ui/update.go
index ca575f2..f4addc3 100644
--- a/internal/ui/update.go
+++ b/internal/ui/update.go
@@ -1,53 +1,54 @@
 package ui
 
 import (
 	"fmt"
 	"time"
 
 	"github.com/charmbracelet/bubbles/list"
 	"github.com/charmbracelet/bubbles/viewport"
 	tea "github.com/charmbracelet/bubbletea"
+	"github.com/dhth/hours/internal/types"
 )
 
 const (
 	viewPortMoveLineCount = 3
+	msgCouldntSelectATask = "Couldn't select a task"
+	msgChangesLocked      = "Changes locked momentarily"
 )
 
-func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
+func (m Model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
 	var cmd tea.Cmd
 	var cmds []tea.Cmd
+
 	m.message = ""
 
-	switch msg := msg.(type) {
-	case tea.KeyMsg:
+	keyMsg, keyMsgOK := msg.(tea.KeyMsg)
+	if keyMsgOK {
 		if m.activeTasksList.FilterState() == list.Filtering {
 			m.activeTasksList, cmd = m.activeTasksList.Update(msg)
 			cmds = append(cmds, cmd)
 			return m, tea.Batch(cmds...)
 		}
-	}
 
-	switch msg := msg.(type) {
-	case tea.KeyMsg:
-		switch msg.String() {
+		switch keyMsg.String() {
 		case "enter":
 			switch m.activeView {
 			case taskInputView:
 				m.activeView = activeTaskListView
 				if m.taskInputs[summaryField].Value() != "" {
 					switch m.taskMgmtContext {
 					case taskCreateCxt:
 						cmds = append(cmds, createTask(m.db, m.taskInputs[summaryField].Value()))
 						m.taskInputs[summaryField].SetValue("")
 					case taskUpdateCxt:
-						selectedTask, ok := m.activeTasksList.SelectedItem().(*task)
+						selectedTask, ok := m.activeTasksList.SelectedItem().(*types.Task)
 						if ok {
 							cmds = append(cmds, updateTask(m.db, selectedTask, m.taskInputs[summaryField].Value()))
 							m.taskInputs[summaryField].SetValue("")
 						}
 					}
 					return m, tea.Batch(cmds...)
 				}
 			case editStartTsView:
 				beginTS, err := time.ParseInLocation(string(timeFormat), m.trackingInputs[entryBeginTS].Value(), time.Local)
 				if err != nil {
@@ -78,21 +79,21 @@ func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
 				if m.activeTLEndTS.Sub(m.activeTLBeginTS).Seconds() <= 0 {
 					m.message = "time spent needs to be positive"
 					return m, tea.Batch(cmds...)
 				}
 
 				if m.trackingInputs[entryComment].Value() == "" {
 					m.message = "Comment cannot be empty"
 					return m, tea.Batch(cmds...)
 				}
 
-				cmds = append(cmds, toggleTracking(m.db, m.activeTaskId, m.activeTLBeginTS, m.activeTLEndTS, m.trackingInputs[entryComment].Value()))
+				cmds = append(cmds, toggleTracking(m.db, m.activeTaskID, m.activeTLBeginTS, m.activeTLEndTS, m.trackingInputs[entryComment].Value()))
 				m.activeView = activeTaskListView
 
 				for i := range m.trackingInputs {
 					m.trackingInputs[i].SetValue("")
 				}
 				return m, tea.Batch(cmds...)
 
 			case manualTasklogEntryView:
 				beginTS, err := time.ParseInLocation(string(timeFormat), m.trackingInputs[entryBeginTS].Value(), time.Local)
 				if err != nil {
@@ -111,49 +112,45 @@ func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
 					return m, tea.Batch(cmds...)
 				}
 
 				comment := m.trackingInputs[entryComment].Value()
 
 				if len(comment) == 0 {
 					m.message = "Comment cannot be empty"
 					return m, tea.Batch(cmds...)
 				}
 
-				task, ok := m.activeTasksList.SelectedItem().(*task)
-				if ok {
-					switch m.tasklogSaveType {
-					case tasklogInsert:
-						cmds = append(cmds, insertManualEntry(m.db, task.id, beginTS, endTS, comment))
-						m.activeView = activeTaskListView
-					}
+				task, ok := m.activeTasksList.SelectedItem().(*types.Task)
+				if ok && m.tasklogSaveType == tasklogInsert {
+					cmds = append(cmds, insertManualEntry(m.db, task.ID, beginTS, endTS, comment))
+					m.activeView = activeTaskListView
 				}
 				for i := range m.trackingInputs {
 					m.trackingInputs[i].SetValue("")
 				}
 				return m, tea.Batch(cmds...)
 			}
 		case "esc":
 			switch m.activeView {
 			case taskInputView:
 				m.activeView = activeTaskListView
 				for i := range m.taskInputs {
 					m.taskInputs[i].SetValue("")
 				}
 			case editStartTsView:
 				m.taskInputs[entryBeginTS].SetValue("")
 				m.activeView = activeTaskListView
 			case askForCommentView:
 				m.activeView = activeTaskListView
 				m.trackingInputs[entryComment].SetValue("")
 			case manualTasklogEntryView:
-				switch m.tasklogSaveType {
-				case tasklogInsert:
+				if m.tasklogSaveType == tasklogInsert {
 					m.activeView = activeTaskListView
 				}
 				for i := range m.trackingInputs {
 					m.trackingInputs[i].SetValue("")
 				}
 			}
 		case "tab":
 			switch m.activeView {
 			case activeTaskListView:
 				m.activeView = taskLogView
@@ -191,36 +188,36 @@ func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
 					m.trackingFocussedField = entryBeginTS
 				case entryComment:
 					m.trackingFocussedField = entryEndTS
 				}
 				for i := range m.trackingInputs {
 					m.trackingInputs[i].Blur()
 				}
 				m.trackingInputs[m.trackingFocussedField].Focus()
 			}
 		case "k":
-			err := m.shiftTime(shiftBackward, shiftMinute)
+			err := m.shiftTime(types.ShiftBackward, types.ShiftMinute)
 			if err != nil {
 				return m, tea.Batch(cmds...)
 			}
 		case "j":
-			err := m.shiftTime(shiftForward, shiftMinute)
+			err := m.shiftTime(types.ShiftForward, types.ShiftMinute)
 			if err != nil {
 				return m, tea.Batch(cmds...)
 			}
 		case "K":
-			err := m.shiftTime(shiftBackward, shiftFiveMinutes)
+			err := m.shiftTime(types.ShiftBackward, types.ShiftFiveMinutes)
 			if err != nil {
 				return m, tea.Batch(cmds...)
 			}
 		case "J":
-			err := m.shiftTime(shiftForward, shiftFiveMinutes)
+			err := m.shiftTime(types.ShiftForward, types.ShiftFiveMinutes)
 			if err != nil {
 				return m, tea.Batch(cmds...)
 			}
 		}
 	}
 
 	switch m.activeView {
 	case taskInputView:
 		for i := range m.taskInputs {
 			m.taskInputs[i], cmd = m.taskInputs[i].Update(msg)
@@ -298,33 +295,33 @@ func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
 			case inactiveTaskListView:
 				cmds = append(cmds, fetchTasks(m.db, false))
 				m.inactiveTasksList.ResetSelected()
 			}
 		case "ctrl+t":
 			if m.activeView == activeTaskListView {
 				if m.trackingActive {
 					if m.activeTasksList.IsFiltered() {
 						m.activeTasksList.ResetFilter()
 					}
-					activeIndex, ok := m.activeTaskIndexMap[m.activeTaskId]
+					activeIndex, ok := m.activeTaskIndexMap[m.activeTaskID]
 					if ok {
 						m.activeTasksList.Select(activeIndex)
 					}
 				} else {
 					m.message = "Nothing is being tracked right now"
 				}
 			}
 		case "ctrl+s":
 			if m.activeView == activeTaskListView {
-				_, ok := m.activeTasksList.SelectedItem().(*task)
+				_, ok := m.activeTasksList.SelectedItem().(*types.Task)
 				if !ok {
-					message := "Couldn't select a task"
+					message := msgCouldntSelectATask
 					m.message = message
 					m.messages = append(m.messages, message)
 				} else {
 					if m.trackingActive {
 						m.activeView = editStartTsView
 						m.trackingFocussedField = entryBeginTS
 						m.trackingInputs[entryBeginTS].SetValue(m.activeTLBeginTS.Format(timeFormat))
 						m.trackingInputs[m.trackingFocussedField].Focus()
 					} else {
 						m.activeView = manualTasklogEntryView
@@ -340,123 +337,120 @@ func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
 						for i := range m.trackingInputs {
 							m.trackingInputs[i].Blur()
 						}
 						m.trackingInputs[m.trackingFocussedField].Focus()
 					}
 				}
 			}
 		case "ctrl+d":
 			switch m.activeView {
 			case activeTaskListView:
-				task, ok := m.activeTasksList.SelectedItem().(*task)
+				task, ok := m.activeTasksList.SelectedItem().(*types.Task)
 				if ok {
-					if task.trackingActive {
+					if task.TrackingActive {
 						m.message = "Cannot deactivate a task being tracked; stop tracking and try again."
 					} else {
 						cmds = append(cmds, updateTaskActiveStatus(m.db, task, false))
 					}
 				} else {
-					msg := "Couldn't select task"
+					msg := msgCouldntSelectATask
 					m.message = msg
 					m.messages = append(m.messages, msg)
 				}
 			case taskLogView:
-				entry, ok := m.taskLogList.SelectedItem().(taskLogEntry)
+				entry, ok := m.taskLogList.SelectedItem().(types.TaskLogEntry)
 				if ok {
 					cmds = append(cmds, deleteLogEntry(m.db, &entry))
 				} else {
 					msg := "Couldn't delete task log entry"
 					m.message = msg
 					m.messages = append(m.messages, msg)
 				}
 			case inactiveTaskListView:
-				task, ok := m.inactiveTasksList.SelectedItem().(*task)
+				task, ok := m.inactiveTasksList.SelectedItem().(*types.Task)
 				if ok {
 					cmds = append(cmds, updateTaskActiveStatus(m.db, task, true))
 				} else {
-					msg := "Couldn't select task"
+					msg := msgCouldntSelectATask
 					m.message = msg
 					m.messages = append(m.messages, msg)
 				}
 			}
 		case "ctrl+x":
 			if m.activeView == activeTaskListView && m.trackingActive {
 				cmds = append(cmds, deleteActiveTaskLog(m.db))
 			}
 		case "s":
-			switch m.activeView {
-			case activeTaskListView:
+			if m.activeView == activeTaskListView {
 				if m.activeTasksList.FilterState() != list.Filtering {
 					if m.changesLocked {
-						message := "Changes locked momentarily"
+						message := msgChangesLocked
 						m.message = message
 						m.messages = append(m.messages, message)
 					}
-					task, ok := m.activeTasksList.SelectedItem().(*task)
+					task, ok := m.activeTasksList.SelectedItem().(*types.Task)
 					if !ok {
 						message := "Couldn't select a task"
 						m.message = message
 						m.messages = append(m.messages, message)
 					} else {
 						if m.lastChange == updateChange {
 							m.changesLocked = true
 							m.activeTLBeginTS = time.Now()
-							cmds = append(cmds, toggleTracking(m.db, task.id, m.activeTLBeginTS, m.activeTLEndTS, ""))
+							cmds = append(cmds, toggleTracking(m.db, task.ID, m.activeTLBeginTS, m.activeTLEndTS, ""))
 						} else if m.lastChange == insertChange {
 							m.activeView = askForCommentView
 							m.activeTLEndTS = time.Now()
 
 							beginTimeStr := m.activeTLBeginTS.Format(timeFormat)
 							currentTimeStr := m.activeTLEndTS.Format(timeFormat)
 
 							m.trackingInputs[entryBeginTS].SetValue(beginTimeStr)
 							m.trackingInputs[entryEndTS].SetValue(currentTimeStr)
 							m.trackingFocussedField = entryComment
 
 							for i := range m.trackingInputs {
 								m.trackingInputs[i].Blur()
 							}
 							m.trackingInputs[m.trackingFocussedField].Focus()
 						}
 					}
 				}
 			}
 		case "a":
-			switch m.activeView {
-			case activeTaskListView:
+			if m.activeView == activeTaskListView {
 				if m.activeTasksList.FilterState() != list.Filtering {
 					if m.changesLocked {
-						message := "Changes locked momentarily"
+						message := msgChangesLocked
 						m.message = message
 						m.messages = append(m.messages, message)
 					}
 					m.activeView = taskInputView
 					m.taskInputFocussedField = summaryField
 					m.taskInputs[summaryField].Focus()
 					m.taskMgmtContext = taskCreateCxt
 				}
 			}
 		case "u":
-			switch m.activeView {
-			case activeTaskListView:
+			if m.activeView == activeTaskListView {
 				if m.activeTasksList.FilterState() != list.Filtering {
 					if m.changesLocked {
-						message := "Changes locked momentarily"
+						message := msgChangesLocked
 						m.message = message
 						m.messages = append(m.messages, message)
 					}
-					task, ok := m.activeTasksList.SelectedItem().(*task)
+					task, ok := m.activeTasksList.SelectedItem().(*types.Task)
 					if ok {
 						m.activeView = taskInputView
 						m.taskInputFocussedField = summaryField
 						m.taskInputs[summaryField].Focus()
-						m.taskInputs[summaryField].SetValue(task.summary)
+						m.taskInputs[summaryField].SetValue(task.Summary)
 						m.taskMgmtContext = taskUpdateCxt
 					} else {
 						m.message = "Couldn't select a task"
 					}
 				}
 			}
 
 		case "k":
 			if m.activeView != helpView {
 				break
@@ -507,49 +501,49 @@ func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
 	case taskCreatedMsg:
 		if msg.err != nil {
 			m.message = fmt.Sprintf("Error creating task: %s", msg.err)
 		} else {
 			cmds = append(cmds, fetchTasks(m.db, true))
 		}
 	case taskUpdatedMsg:
 		if msg.err != nil {
 			m.message = fmt.Sprintf("Error updating task: %s", msg.err)
 		} else {
-			msg.tsk.summary = msg.summary
-			msg.tsk.updateTitle()
+			msg.tsk.Summary = msg.summary
+			msg.tsk.UpdateTitle()
 		}
 	case tasksFetched:
 		if msg.err != nil {
 			message := "error fetching tasks : " + msg.err.Error()
 			m.message = message
 			m.messages = append(m.messages, message)
 		} else {
 			if msg.active {
-				m.activeTaskMap = make(map[int]*task)
+				m.activeTaskMap = make(map[int]*types.Task)
 				m.activeTaskIndexMap = make(map[int]int)
 				tasks := make([]list.Item, len(msg.tasks))
 				for i, task := range msg.tasks {
-					task.updateTitle()
-					task.updateDesc()
+					task.UpdateTitle()
+					task.UpdateDesc()
 					tasks[i] = &task
-					m.activeTaskMap[task.id] = &task
-					m.activeTaskIndexMap[task.id] = i
+					m.activeTaskMap[task.ID] = &task
+					m.activeTaskIndexMap[task.ID] = i
 				}
 				m.activeTasksList.SetItems(tasks)
 				m.activeTasksList.Title = "Tasks"
 				m.tasksFetched = true
 				cmds = append(cmds, fetchActiveTask(m.db))
 			} else {
 				inactiveTasks := make([]list.Item, len(msg.tasks))
 				for i, inactiveTask := range msg.tasks {
-					inactiveTask.updateTitle()
-					inactiveTask.updateDesc()
+					inactiveTask.UpdateTitle()
+					inactiveTask.UpdateDesc()
 					inactiveTasks[i] = &inactiveTask
 				}
 				m.inactiveTasksList.SetItems(inactiveTasks)
 			}
 		}
 	case tlBeginTSUpdatedMsg:
 		if msg.err != nil {
 			message := msg.err.Error()
 			m.message = "Error updating begin time: " + message
 			m.messages = append(m.messages, message)
@@ -558,125 +552,125 @@ func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
 		}
 	case manualTaskLogInserted:
 		if msg.err != nil {
 			message := msg.err.Error()
 			m.message = "Error inserting task log: " + message
 			m.messages = append(m.messages, message)
 		} else {
 			for i := range m.trackingInputs {
 				m.trackingInputs[i].SetValue("")
 			}
-			task, ok := m.activeTaskMap[msg.taskId]
+			task, ok := m.activeTaskMap[msg.taskID]
 
 			if ok {
 				cmds = append(cmds, updateTaskRep(m.db, task))
 			}
 			cmds = append(cmds, fetchTaskLogEntries(m.db))
 		}
 	case taskLogEntriesFetchedMsg:
 		if msg.err != nil {
 			message := msg.err.Error()
 			m.message = "Error fetching task log entries: " + message
 			m.messages = append(m.messages, message)
 		} else {
 			var items []list.Item
 			for _, e := range msg.entries {
-				e.updateTitle()
-				e.updateDesc()
+				e.UpdateTitle()
+				e.UpdateDesc()
 				items = append(items, e)
 			}
 			m.taskLogList.SetItems(items)
 		}
 	case activeTaskFetchedMsg:
 		if msg.err != nil {
 			message := msg.err.Error()
 			m.message = message
 			m.messages = append(m.messages, message)
 		} else {
 			if msg.noneActive {
 				m.lastChange = updateChange
 			} else {
-				m.activeTaskId = msg.activeTaskId
+				m.activeTaskID = msg.activeTaskID
 				m.lastChange = insertChange
 				m.activeTLBeginTS = msg.beginTs
-				activeTask, ok := m.activeTaskMap[m.activeTaskId]
+				activeTask, ok := m.activeTaskMap[m.activeTaskID]
 				if ok {
-					activeTask.trackingActive = true
-					activeTask.updateTitle()
+					activeTask.TrackingActive = true
+					activeTask.UpdateTitle()
 
 					// go to tracked item on startup
-					activeIndex, ok := m.activeTaskIndexMap[msg.activeTaskId]
+					activeIndex, ok := m.activeTaskIndexMap[msg.activeTaskID]
 					if ok {
 						m.activeTasksList.Select(activeIndex)
 					}
 				}
 				m.trackingActive = true
 			}
 		}
 	case trackingToggledMsg:
 		if msg.err != nil {
 			message := msg.err.Error()
 			m.message = message
 			m.messages = append(m.messages, message)
 			m.trackingActive = false
 		} else {
 			m.changesLocked = false
 
-			task, ok := m.activeTaskMap[msg.taskId]
+			task, ok := m.activeTaskMap[msg.taskID]
 
 			if ok {
 				if msg.finished {
 					m.lastChange = updateChange
-					task.trackingActive = false
+					task.TrackingActive = false
 					m.trackingActive = false
-					m.activeTaskId = -1
+					m.activeTaskID = -1
 					cmds = append(cmds, updateTaskRep(m.db, task))
 					cmds = append(cmds, fetchTaskLogEntries(m.db))
 				} else {
 					m.lastChange = insertChange
-					task.trackingActive = true
+					task.TrackingActive = true
 					m.trackingActive = true
-					m.activeTaskId = msg.taskId
+					m.activeTaskID = msg.taskID
 				}
-				task.updateTitle()
+				task.UpdateTitle()
 			}
 		}
 	case taskRepUpdatedMsg:
 		if msg.err != nil {
 			m.message = fmt.Sprintf("Error updating task status: %s", msg.err)
 		} else {
-			msg.tsk.updateDesc()
+			msg.tsk.UpdateDesc()
 		}
 	case taskLogEntryDeletedMsg:
 		if msg.err != nil {
 			message := "error deleting entry: " + msg.err.Error()
 			m.message = message
 			m.messages = append(m.messages, message)
 		} else {
-			task, ok := m.activeTaskMap[msg.entry.taskId]
+			task, ok := m.activeTaskMap[msg.entry.TaskID]
 			if ok {
 				cmds = append(cmds, updateTaskRep(m.db, task))
 			}
 			cmds = append(cmds, fetchTaskLogEntries(m.db))
 		}
 	case activeTaskLogDeletedMsg:
 		if msg.err != nil {
 			m.message = fmt.Sprintf("Error deleting active log entry: %s", msg.err)
 		} else {
-			activeTask, ok := m.activeTaskMap[m.activeTaskId]
+			activeTask, ok := m.activeTaskMap[m.activeTaskID]
 			if ok {
-				activeTask.trackingActive = false
-				activeTask.updateTitle()
+				activeTask.TrackingActive = false
+				activeTask.UpdateTitle()
 			}
 			m.lastChange = updateChange
 			m.trackingActive = false
-			m.activeTaskId = -1
+			m.activeTaskID = -1
 		}
 	case taskActiveStatusUpdated:
 		if msg.err != nil {
 			message := "error updating task's active status: " + msg.err.Error()
 			m.message = message
 			m.messages = append(m.messages, message)
 		} else {
 			cmds = append(cmds, fetchTasks(m.db, true))
 			cmds = append(cmds, fetchTasks(m.db, false))
 		}
@@ -709,62 +703,62 @@ func (m recordsModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
 		switch msg.String() {
 		case "ctrl+c", "q":
 			m.quitting = true
 			return m, tea.Quit
 		case "left", "h":
 			if !m.busy {
 				var newStart, newEnd time.Time
 				var numDays int
 
 				switch m.period {
-				case "week":
+				case types.TimePeriodWeek:
 					weekday := m.start.Weekday()
 					offset := (7 + weekday - time.Monday) % 7
 					startOfPrevWeek := m.start.AddDate(0, 0, -int(offset+7))
 					newStart = time.Date(startOfPrevWeek.Year(), startOfPrevWeek.Month(), startOfPrevWeek.Day(), 0, 0, 0, 0, startOfPrevWeek.Location())
 					numDays = 7
 				default:
 					newStart = m.start.AddDate(0, 0, -m.numDays)
 					numDays = m.numDays
 				}
 				newEnd = newStart.AddDate(0, 0, numDays)
 				cmds = append(cmds, getRecordsData(m.typ, m.db, m.period, newStart, newEnd, numDays, m.plain))
 				m.busy = true
 			}
 		case "right", "l":
 			if !m.busy {
 				var newStart, newEnd time.Time
 				var numDays int
 
 				switch m.period {
-				case "week":
+				case types.TimePeriodWeek:
 					weekday := m.start.Weekday()
 					offset := (7 + weekday - time.Monday) % 7
 					startOfNextWeek := m.start.AddDate(0, 0, 7-int(offset))
 					newStart = time.Date(startOfNextWeek.Year(), startOfNextWeek.Month(), startOfNextWeek.Day(), 0, 0, 0, 0, startOfNextWeek.Location())
 					numDays = 7
 
 				default:
 					newStart = m.start.AddDate(0, 0, 1*(m.numDays))
 					numDays = m.numDays
 				}
 				newEnd = newStart.AddDate(0, 0, numDays)
 				cmds = append(cmds, getRecordsData(m.typ, m.db, m.period, newStart, newEnd, numDays, m.plain))
 				m.busy = true
 			}
 		case "ctrl+t":
 			if !m.busy {
 				var start, end time.Time
 				var numDays int
 
 				switch m.period {
-				case "week":
+				case types.TimePeriodWeek:
 					now := time.Now()
 					weekday := now.Weekday()
 					offset := (7 + weekday - time.Monday) % 7
 					startOfWeek := now.AddDate(0, 0, -int(offset))
 					start = time.Date(startOfWeek.Year(), startOfWeek.Month(), startOfWeek.Day(), 0, 0, 0, 0, startOfWeek.Location())
 					numDays = 7
 				default:
 					now := time.Now()
 					nDaysBack := now.AddDate(0, 0, -1*(m.numDays-1))
 
@@ -784,25 +778,25 @@ func (m recordsModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
 		} else {
 			m.start = msg.start
 			m.end = msg.end
 			m.report = msg.report
 			m.busy = false
 		}
 	}
 	return m, tea.Batch(cmds...)
 }
 
-func (m model) shiftTime(direction timeShiftDirection, duration timeShiftDuration) error {
+func (m Model) shiftTime(direction types.TimeShiftDirection, duration types.TimeShiftDuration) error {
 	if m.activeView == editStartTsView || m.activeView == askForCommentView || m.activeView == manualTasklogEntryView {
 		if m.trackingFocussedField == entryBeginTS || m.trackingFocussedField == entryEndTS {
 			ts, err := time.ParseInLocation(string(timeFormat), m.trackingInputs[m.trackingFocussedField].Value(), time.Local)
 			if err != nil {
 				return err
 			}
 
-			newTs := getShiftedTime(ts, direction, duration)
+			newTs := types.GetShiftedTime(ts, direction, duration)
 
 			m.trackingInputs[m.trackingFocussedField].SetValue(newTs.Format(timeFormat))
 		}
 	}
 	return nil
 }
diff --git a/internal/ui/utils.go b/internal/ui/utils.go
index 1816a08..d4fe366 100644
--- a/internal/ui/utils.go
+++ b/internal/ui/utils.go
@@ -1,67 +1,22 @@
 package ui
 
 import (
-	"fmt"
-	"math"
-	"strings"
-	"time"
-
 	"github.com/charmbracelet/lipgloss"
 )
 
 type reportStyles struct {
 	headerStyle lipgloss.Style
 	footerStyle lipgloss.Style
 	borderStyle lipgloss.Style
 }
 
-func RightPadTrim(s string, length int, dots bool) string {
-	if len(s) >= length {
-		if dots && length > 3 {
-			return s[:length-3] + "..."
-		}
-		return s[:length]
-	}
-	return s + strings.Repeat(" ", length-len(s))
-}
-
-func Trim(s string, length int) string {
-	if len(s) >= length {
-		if length > 3 {
-			return s[:length-3] + "..."
-		}
-		return s[:length]
-	}
-	return s
-}
-
-func humanizeDuration(durationInSecs int) string {
-	duration := time.Duration(durationInSecs) * time.Second
-
-	if duration.Seconds() < 60 {
-		return fmt.Sprintf("%ds", int(duration.Seconds()))
-	}
-
-	if duration.Minutes() < 60 {
-		return fmt.Sprintf("%dm", int(duration.Minutes()))
-	}
-
-	modMins := int(math.Mod(duration.Minutes(), 60))
-
-	if modMins == 0 {
-		return fmt.Sprintf("%dh", int(duration.Hours()))
-	}
-
-	return fmt.Sprintf("%dh %dm", int(duration.Hours()), modMins)
-}
-
 func getReportStyles(plain bool) reportStyles {
 	if plain {
 		return reportStyles{
 			emptyStyle,
 			emptyStyle,
 			emptyStyle,
 		}
 	}
 	return reportStyles{
 		recordsHeaderStyle,
diff --git a/internal/ui/view.go b/internal/ui/view.go
index 1726016..9121577 100644
--- a/internal/ui/view.go
+++ b/internal/ui/view.go
@@ -1,39 +1,40 @@
 package ui
 
 import (
 	"fmt"
 
 	"github.com/charmbracelet/lipgloss"
+	"github.com/dhth/hours/internal/utils"
 )
 
 const (
 	taskLogEntryViewHeading = "Task Log Entry"
 )
 
 var listWidth = 140
 
-func (m model) View() string {
+func (m Model) View() string {
 	var content string
 	var footer string
 
 	var statusBar string
 	if m.message != "" {
-		statusBar = Trim(m.message, 120)
+		statusBar = utils.Trim(m.message, 120)
 	}
 
 	var activeMsg string
 	if m.tasksFetched && m.trackingActive {
 		var taskSummaryMsg, taskStartedSinceMsg string
-		task, ok := m.activeTaskMap[m.activeTaskId]
+		task, ok := m.activeTaskMap[m.activeTaskID]
 		if ok {
-			taskSummaryMsg = Trim(task.summary, 50)
+			taskSummaryMsg = utils.Trim(task.Summary, 50)
 			if m.activeView != askForCommentView {
 				taskStartedSinceMsg = fmt.Sprintf("(since %s)", m.activeTLBeginTS.Format(timeOnlyFormat))
 			}
 		}
 		activeMsg = fmt.Sprintf("%s%s%s",
 			trackingStyle.Render("tracking:"),
 			activeTaskSummaryMsgStyle.Render(taskSummaryMsg),
 			activeTaskBeginTimeStyle.Render(taskStartedSinceMsg),
 		)
 	}
@@ -97,21 +98,21 @@ func (m model) View() string {
 `,
 			taskLogEntryHeadingStyle.Render(taskLogEntryViewHeading),
 			formContextStyle.Render(formHeadingText),
 			formHelpStyle.Render("Use tab/shift-tab to move between sections; esc to go back."),
 			formFieldNameStyle.Render("Begin Time (format: 2006/01/02 15:04)"),
 			m.trackingInputs[entryBeginTS].View(),
 			formHelpStyle.Render("(k/j/K/J moves time, when correct)"),
 			formFieldNameStyle.Render("End Time (format: 2006/01/02 15:04)"),
 			m.trackingInputs[entryEndTS].View(),
 			formHelpStyle.Render("(k/j/K/J moves time, when correct)"),
-			formFieldNameStyle.Render(RightPadTrim("Comment:", 16, true)),
+			formFieldNameStyle.Render(utils.RightPadTrim("Comment:", 16, true)),
 			m.trackingInputs[entryComment].View(),
 			formHelpStyle.Render("Press enter to submit"),
 		)
 		for i := 0; i < m.terminalHeight-24; i++ {
 			content += "\n"
 		}
 	case editStartTsView:
 		formHeadingText := "Updating log entry. Enter the following details."
 
 		content = fmt.Sprintf(
@@ -171,21 +172,21 @@ func (m model) View() string {
 `,
 			taskLogEntryHeadingStyle.Render(taskLogEntryViewHeading),
 			formContextStyle.Render(formHeadingText),
 			formHelpStyle.Render("Use tab/shift-tab to move between sections; esc to go back."),
 			formFieldNameStyle.Render("Begin Time (format: 2006/01/02 15:04)"),
 			m.trackingInputs[entryBeginTS].View(),
 			formHelpStyle.Render("(k/j/K/J moves time, when correct)"),
 			formFieldNameStyle.Render("End Time (format: 2006/01/02 15:04)"),
 			m.trackingInputs[entryEndTS].View(),
 			formHelpStyle.Render("(k/j/K/J moves time, when correct)"),
-			formFieldNameStyle.Render(RightPadTrim("Comment:", 16, true)),
+			formFieldNameStyle.Render(utils.RightPadTrim("Comment:", 16, true)),
 			m.trackingInputs[entryComment].View(),
 			formHelpStyle.Render("Press enter to submit"),
 		)
 		for i := 0; i < m.terminalHeight-24; i++ {
 			content += "\n"
 		}
 	case helpView:
 		if !m.helpVPReady {
 			content = "\n  Initializing..."
 		} else {
diff --git a/internal/utils/utils.go b/internal/utils/utils.go
new file mode 100644
index 0000000..afec04b
--- /dev/null
+++ b/internal/utils/utils.go
@@ -0,0 +1,23 @@
+package utils
+
+import "strings"
+
+func RightPadTrim(s string, length int, dots bool) string {
+	if len(s) >= length {
+		if dots && length > 3 {
+			return s[:length-3] + "..."
+		}
+		return s[:length]
+	}
+	return s + strings.Repeat(" ", length-len(s))
+}
+
+func Trim(s string, length int) string {
+	if len(s) >= length {
+		if length > 3 {
+			return s[:length-3] + "..."
+		}
+		return s[:length]
+	}
+	return s
+}
diff --git a/main.go b/main.go
index 3fc0abd..59a81c0 100644
--- a/main.go
+++ b/main.go
@@ -1,7 +1,14 @@
 package main
 
-import "github.com/dhth/hours/cmd"
+import (
+	"os"
+
+	"github.com/dhth/hours/cmd"
+)
 
 func main() {
-	cmd.Execute()
+	err := cmd.Execute()
+	if err != nil {
+		os.Exit(1)
+	}
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

👉 The "distilled" version of the same diff looks like the following:

<details><summary> expand </summary>

```diff
diff --git a/ac7d86e/cmd/db.go b/ac7d86e/cmd/db.go
deleted file mode 100644
index 3e1ef28..0000000
--- a/ac7d86e/cmd/db.go
+++ /dev/null
@@ -1,2 +0,0 @@
-func getDB(dbpath string) (*sql.DB, error)
-func initDB(db *sql.DB) error
diff --git a/ac7d86e/cmd/db_migrations_test.go b/ac7d86e/cmd/db_migrations_test.go
deleted file mode 100644
index 72d6fbd..0000000
--- a/ac7d86e/cmd/db_migrations_test.go
+++ /dev/null
@@ -1 +0,0 @@
-func TestMigrationsAreSetupCorrectly(t *testing.T)
diff --git a/ac7d86e/cmd/root.go b/53b6ad1/cmd/root.go
index 92552b7..2dd1453 100644
--- a/ac7d86e/cmd/root.go
+++ b/53b6ad1/cmd/root.go
@@ -1,4 +1,3 @@
-func die(msg string, args ...any)
-func setupDB()
-func init()
-func Execute()
+func Execute() error
+func setupDB(dbPathFull string) (*sql.DB, error)
+func NewRootCommand() (*cobra.Command, error)
@@ -6 +5 @@ func getRandomChars(length int) string
-func getConfirmation() bool
+func getConfirmation() (bool, error)
diff --git a/ac7d86e/cmd/utils.go b/53b6ad1/cmd/utils.go
index 9dc7039..9127755 100644
--- a/ac7d86e/cmd/utils.go
+++ b/53b6ad1/cmd/utils.go
@@ -1 +1 @@
-func expandTilde(path string) string
+func expandTilde(path string, homeDir string) string
diff --git a/53b6ad1/cmd/utils_test.go b/53b6ad1/cmd/utils_test.go
new file mode 100644
index 0000000..ce52ef3
--- /dev/null
+++ b/53b6ad1/cmd/utils_test.go
@@ -0,0 +1 @@
+func TestExpandTilde(t *testing.T)
diff --git a/53b6ad1/internal/persistence/init.go b/53b6ad1/internal/persistence/init.go
new file mode 100644
index 0000000..fa4accf
--- /dev/null
+++ b/53b6ad1/internal/persistence/init.go
@@ -0,0 +1 @@
+func InitDB(db *sql.DB) error
diff --git a/ac7d86e/cmd/db_migrations.go b/53b6ad1/internal/persistence/migrations.go
similarity index 72%
rename from ac7d86e/cmd/db_migrations.go
rename to 53b6ad1/internal/persistence/migrations.go
index 1ada63d..654706a 100644
--- a/ac7d86e/cmd/db_migrations.go
+++ b/53b6ad1/internal/persistence/migrations.go
@@ -8,2 +8,2 @@ func fetchLatestDBVersion(db *sql.DB) (dbVersionInfo, error)
-func upgradeDBIfNeeded(db *sql.DB)
-func upgradeDB(db *sql.DB, currentVersion int)
+func UpgradeDBIfNeeded(db *sql.DB) error
+func UpgradeDB(db *sql.DB, currentVersion int) error
diff --git a/53b6ad1/internal/persistence/migrations_test.go b/53b6ad1/internal/persistence/migrations_test.go
new file mode 100644
index 0000000..9b71efb
--- /dev/null
+++ b/53b6ad1/internal/persistence/migrations_test.go
@@ -0,0 +1,3 @@
+func TestMigrationsAreSetupCorrectly(t *testing.T)
+func TestMigrationsWork(t *testing.T)
+func TestRunMigrationFailsWhenGivenBadMigration(t *testing.T)
diff --git a/53b6ad1/internal/persistence/open.go b/53b6ad1/internal/persistence/open.go
new file mode 100644
index 0000000..e2b6962
--- /dev/null
+++ b/53b6ad1/internal/persistence/open.go
@@ -0,0 +1 @@
+func GetDB(dbpath string) (*sql.DB, error)
diff --git a/53b6ad1/internal/persistence/queries.go b/53b6ad1/internal/persistence/queries.go
new file mode 100644
index 0000000..e6711eb
--- /dev/null
+++ b/53b6ad1/internal/persistence/queries.go
@@ -0,0 +1,21 @@
+func InsertNewTL(db *sql.DB, taskID int, beginTs time.Time) (int, error)
+func UpdateTLBeginTS(db *sql.DB, beginTs time.Time) error
+func DeleteActiveTL(db *sql.DB) error
+func UpdateActiveTL(db *sql.DB, taskLogID int, taskID int, beginTs, endTs time.Time, secsSpent int, comment string) error
+func InsertManualTL(db *sql.DB, taskID int, beginTs time.Time, endTs time.Time, comment string) (int, error)
+func FetchActiveTask(db *sql.DB) (types.ActiveTaskDetails, error)
+func InsertTask(db *sql.DB, summary string) (int, error)
+func UpdateTask(db *sql.DB, id int, summary string) error
+func UpdateTaskActiveStatus(db *sql.DB, id int, active bool) error
+func UpdateTaskData(db *sql.DB, t *types.Task) error
+func FetchTasks(db *sql.DB, active bool, limit int) ([]types.Task, error)
+func FetchTLEntries(db *sql.DB, desc bool, limit int) ([]types.TaskLogEntry, error)
+func FetchTLEntriesBetweenTS(db *sql.DB, beginTs, endTs time.Time, limit int) ([]types.TaskLogEntry, error)
+func FetchStats(db *sql.DB, limit int) ([]types.TaskReportEntry, error)
+func FetchStatsBetweenTS(db *sql.DB, beginTs, endTs time.Time, limit int) ([]types.TaskReportEntry, error)
+func FetchReportBetweenTS(db *sql.DB, beginTs, endTs time.Time, limit int) ([]types.TaskReportEntry, error)
+func DeleteTaskLogEntry(db *sql.DB, entry *types.TaskLogEntry) error
+func runInTxAndReturnID(db *sql.DB, fn func(tx *sql.Tx) (int, error)) (int, error)
+func runInTx(db *sql.DB, fn func(tx *sql.Tx) error) error
+func fetchTaskByID(db *sql.DB, id int) (types.Task, error)
+func fetchTaskLogByID(db *sql.DB, id int) (types.TaskLogEntry, error)
diff --git a/53b6ad1/internal/persistence/queries_test.go b/53b6ad1/internal/persistence/queries_test.go
new file mode 100644
index 0000000..50c2dd1
--- /dev/null
+++ b/53b6ad1/internal/persistence/queries_test.go
@@ -0,0 +1,8 @@
+type testData struct {
+	tasks    []types.Task
+	taskLogs []types.TaskLogEntry
+}
+func TestRepository(t *testing.T)
+func cleanupDB(t *testing.T, testDB *sql.DB)
+func getTestData(referenceTS time.Time) testData
+func seedDB(t *testing.T, db *sql.DB, data testData)
diff --git a/53b6ad1/internal/types/date_helpers.go b/53b6ad1/internal/types/date_helpers.go
new file mode 100644
index 0000000..29f70d0
--- /dev/null
+++ b/53b6ad1/internal/types/date_helpers.go
@@ -0,0 +1,10 @@
+type TimePeriod struct {
+	Start   time.Time
+	End     time.Time
+	NumDays int
+}
+type tsRelative uint8
+func parseDateDuration(dateRange string) (TimePeriod, error)
+func GetTimePeriod(period string, now time.Time, fullWeek bool) (TimePeriod, error)
+func GetShiftedTime(ts time.Time, direction TimeShiftDirection, duration TimeShiftDuration) time.Time
+func getTSRelative(ts time.Time, reference time.Time) tsRelative
diff --git a/ac7d86e/internal/ui/date_helpers_test.go b/53b6ad1/internal/types/date_helpers_test.go
similarity index 100%
rename from ac7d86e/internal/ui/date_helpers_test.go
rename to 53b6ad1/internal/types/date_helpers_test.go
diff --git a/53b6ad1/internal/types/types.go b/53b6ad1/internal/types/types.go
new file mode 100644
index 0000000..0731bd1
--- /dev/null
+++ b/53b6ad1/internal/types/types.go
@@ -0,0 +1,46 @@
+type Task struct {
+	ID             int
+	Summary        string
+	CreatedAt      time.Time
+	UpdatedAt      time.Time
+	TrackingActive bool
+	SecsSpent      int
+	Active         bool
+	TaskTitle      string
+	TaskDesc       string
+}
+type TaskLogEntry struct {
+	ID          int
+	TaskID      int
+	TaskSummary string
+	BeginTS     time.Time
+	EndTS       time.Time
+	SecsSpent   int
+	Comment     string
+	TLTitle     string
+	TLDesc      string
+}
+type ActiveTaskDetails struct {
+	TaskID              int
+	TaskSummary         string
+	LastLogEntryBeginTS time.Time
+}
+type TaskReportEntry struct {
+	TaskID      int
+	TaskSummary string
+	NumEntries  int
+	SecsSpent   int
+}
+type TimeShiftDirection uint8
+type TimeShiftDuration uint8
+func HumanizeDuration(durationInSecs int) string
+func (t *Task) UpdateTitle()
+func (t *Task) UpdateDesc()
+func (tl *TaskLogEntry) UpdateTitle()
+func (tl *TaskLogEntry) UpdateDesc()
+func (t Task) Title() string
+func (t Task) Description() string
+func (t Task) FilterValue() string
+func (tl TaskLogEntry) Title() string
+func (tl TaskLogEntry) Description() string
+func (tl TaskLogEntry) FilterValue() string
diff --git a/53b6ad1/internal/types/types_test.go b/53b6ad1/internal/types/types_test.go
new file mode 100644
index 0000000..9d3c315
--- /dev/null
+++ b/53b6ad1/internal/types/types_test.go
@@ -0,0 +1 @@
+func TestHumanizeDuration(t *testing.T)
diff --git a/ac7d86e/internal/ui/active.go b/53b6ad1/internal/ui/active.go
index 3a476ba..7c31fa2 100644
--- a/ac7d86e/internal/ui/active.go
+++ b/53b6ad1/internal/ui/active.go
@@ -1 +1 @@
-func ShowActiveTask(db *sql.DB, writer io.Writer, template string)
+func ShowActiveTask(db *sql.DB, writer io.Writer, template string) error
diff --git a/ac7d86e/internal/ui/cmds.go b/53b6ad1/internal/ui/cmds.go
index e34d42c..16a5119 100644
--- a/ac7d86e/internal/ui/cmds.go
+++ b/53b6ad1/internal/ui/cmds.go
@@ -2 +2 @@ func toggleTracking(db *sql.DB,
-	taskId int,
+	taskID int,
@@ -8 +8 @@ func updateTLBeginTS(db *sql.DB, beginTS time.Time) tea.Cmd
-func insertManualEntry(db *sql.DB, taskId int, beginTS time.Time, endTS time.Time, comment string) tea.Cmd
+func insertManualEntry(db *sql.DB, taskID int, beginTS time.Time, endTS time.Time, comment string) tea.Cmd
@@ -10 +10 @@ func fetchActiveTask(db *sql.DB) tea.Cmd
-func updateTaskRep(db *sql.DB, t *task) tea.Cmd
+func updateTaskRep(db *sql.DB, t *types.Task) tea.Cmd
@@ -12 +12 @@ func fetchTaskLogEntries(db *sql.DB) tea.Cmd
-func deleteLogEntry(db *sql.DB, entry *taskLogEntry) tea.Cmd
+func deleteLogEntry(db *sql.DB, entry *types.TaskLogEntry) tea.Cmd
@@ -15,2 +15,2 @@ func createTask(db *sql.DB, summary string) tea.Cmd
-func updateTask(db *sql.DB, task *task, summary string) tea.Cmd
-func updateTaskActiveStatus(db *sql.DB, task *task, active bool) tea.Cmd
+func updateTask(db *sql.DB, task *types.Task, summary string) tea.Cmd
+func updateTaskActiveStatus(db *sql.DB, task *types.Task, active bool) tea.Cmd
diff --git a/ac7d86e/internal/ui/date_helpers.go b/ac7d86e/internal/ui/date_helpers.go
deleted file mode 100644
index b41d9d2..0000000
--- a/ac7d86e/internal/ui/date_helpers.go
+++ /dev/null
@@ -1,12 +0,0 @@
-type timePeriod struct {
-	start   time.Time
-	end     time.Time
-	numDays int
-}
-type timeShiftDirection uint8
-type timeShiftDuration uint8
-type tsRelative uint8
-func parseDateDuration(dateRange string) (timePeriod, bool)
-func getTimePeriod(period string, now time.Time, fullWeek bool) (timePeriod, error)
-func getShiftedTime(ts time.Time, direction timeShiftDirection, duration timeShiftDuration) time.Time
-func getTSRelative(ts time.Time, reference time.Time) tsRelative
diff --git a/ac7d86e/internal/ui/initial.go b/53b6ad1/internal/ui/initial.go
index 544b630..4d99dc0 100644
--- a/ac7d86e/internal/ui/initial.go
+++ b/53b6ad1/internal/ui/initial.go
@@ -1 +1 @@
-func InitialModel(db *sql.DB) model
+func InitialModel(db *sql.DB) Model
diff --git a/ac7d86e/internal/ui/log.go b/53b6ad1/internal/ui/log.go
index 96fb1e0..3a914df 100644
--- a/ac7d86e/internal/ui/log.go
+++ b/53b6ad1/internal/ui/log.go
@@ -1,2 +1,2 @@
-func RenderTaskLog(db *sql.DB, writer io.Writer, plain bool, period string, interactive bool)
-func renderTaskLog(db *sql.DB, start, end time.Time, limit int, plain bool) (string, error)
+func RenderTaskLog(db *sql.DB, writer io.Writer, plain bool, period string, interactive bool) error
+func getTaskLog(db *sql.DB, start, end time.Time, limit int, plain bool) (string, error)
diff --git a/ac7d86e/internal/ui/model.go b/53b6ad1/internal/ui/model.go
index 8050cc3..bc734f4 100644
--- a/ac7d86e/internal/ui/model.go
+++ b/53b6ad1/internal/ui/model.go
@@ -9 +9 @@ type recordsType uint
-type model struct {
+type Model struct {
@@ -15 +15 @@ type model struct {
-	activeTaskMap          map[int]*task
+	activeTaskMap          map[int]*types.Task
@@ -30 +30 @@ type model struct {
-	activeTaskId           int
+	activeTaskID           int
@@ -51,2 +51,2 @@ type recordsModel struct {
-func (m model) Init() tea.Cmd
-func (m recordsModel) Init() tea.Cmd
+func (m Model) Init() tea.Cmd
+func (recordsModel) Init() tea.Cmd
diff --git a/ac7d86e/internal/ui/msgs.go b/53b6ad1/internal/ui/msgs.go
index 5b11545..b032a77 100644
--- a/ac7d86e/internal/ui/msgs.go
+++ b/53b6ad1/internal/ui/msgs.go
@@ -3 +3 @@ type trackingToggledMsg struct {
-	taskId    int
+	taskID    int
@@ -9 +9 @@ type taskRepUpdatedMsg struct {
-	tsk *task
+	tsk *types.Task
@@ -13 +13 @@ type manualTaskLogInserted struct {
-	taskId int
+	taskID int
@@ -24 +24 @@ type activeTaskFetchedMsg struct {
-	activeTaskId int
+	activeTaskID int
@@ -30 +30 @@ type taskLogEntriesFetchedMsg struct {
-	entries []taskLogEntry
+	entries []types.TaskLogEntry
@@ -37 +37 @@ type taskUpdatedMsg struct {
-	tsk     *task
+	tsk     *types.Task
@@ -42 +42 @@ type taskActiveStatusUpdated struct {
-	tsk    *task
+	tsk    *types.Task
@@ -47 +47 @@ type taskLogEntryDeletedMsg struct {
-	entry *taskLogEntry
+	entry *types.TaskLogEntry
@@ -51 +51 @@ type tasksFetched struct {
-	tasks  []task
+	tasks  []types.Task
diff --git a/ac7d86e/internal/ui/queries.go b/ac7d86e/internal/ui/queries.go
deleted file mode 100644
index 6fa7bbd..0000000
--- a/ac7d86e/internal/ui/queries.go
+++ /dev/null
@@ -1,17 +0,0 @@
-func insertNewTLInDB(db *sql.DB, taskId int, beginTs time.Time) error
-func updateTLBeginTSInDB(db *sql.DB, beginTs time.Time) error
-func deleteActiveTLInDB(db *sql.DB) error
-func updateActiveTLInDB(db *sql.DB, taskLogId int, taskId int, beginTs, endTs time.Time, secsSpent int, comment string) error
-func insertManualTLInDB(db *sql.DB, taskId int, beginTs time.Time, endTs time.Time, comment string) error
-func fetchActiveTaskFromDB(db *sql.DB) (activeTaskDetails, error)
-func insertTaskInDB(db *sql.DB, summary string) error
-func updateTaskInDB(db *sql.DB, id int, summary string) error
-func updateTaskActiveStatusInDB(db *sql.DB, id int, active bool) error
-func updateTaskDataFromDB(db *sql.DB, t *task) error
-func fetchTasksFromDB(db *sql.DB, active bool, limit int) ([]task, error)
-func fetchTLEntriesFromDB(db *sql.DB, desc bool, limit int) ([]taskLogEntry, error)
-func fetchTLEntriesBetweenTSFromDB(db *sql.DB, beginTs, endTs time.Time, limit int) ([]taskLogEntry, error)
-func fetchStatsFromDB(db *sql.DB, limit int) ([]taskReportEntry, error)
-func fetchStatsBetweenTSFromDB(db *sql.DB, beginTs, endTs time.Time, limit int) ([]taskReportEntry, error)
-func fetchReportBetweenTSFromDB(db *sql.DB, beginTs, endTs time.Time, limit int) ([]taskReportEntry, error)
-func deleteEntry(db *sql.DB, entry *taskLogEntry) error
diff --git a/ac7d86e/internal/ui/render_helpers.go b/ac7d86e/internal/ui/render_helpers.go
deleted file mode 100644
index cbd4992..0000000
--- a/ac7d86e/internal/ui/render_helpers.go
+++ /dev/null
@@ -1,4 +0,0 @@
-func (t *task) updateTitle()
-func (t *task) updateDesc()
-func (tl *taskLogEntry) updateTitle()
-func (tl *taskLogEntry) updateDesc()
diff --git a/ac7d86e/internal/ui/report.go b/53b6ad1/internal/ui/report.go
index c6602d1..932dc90 100644
--- a/ac7d86e/internal/ui/report.go
+++ b/53b6ad1/internal/ui/report.go
@@ -1 +1 @@
-func RenderReport(db *sql.DB, writer io.Writer, plain bool, period string, agg bool, interactive bool)
+func RenderReport(db *sql.DB, writer io.Writer, plain bool, period string, agg bool, interactive bool) error
diff --git a/ac7d86e/internal/ui/stats.go b/53b6ad1/internal/ui/stats.go
index b52b202..8b61adb 100644
--- a/ac7d86e/internal/ui/stats.go
+++ b/53b6ad1/internal/ui/stats.go
@@ -1,2 +1,2 @@
-func RenderStats(db *sql.DB, writer io.Writer, plain bool, period string, interactive bool)
-func renderStats(db *sql.DB, period string, start, end time.Time, plain bool) (string, error)
+func RenderStats(db *sql.DB, writer io.Writer, plain bool, period string, interactive bool) error
+func getStats(db *sql.DB, period string, start, end time.Time, plain bool) (string, error)
diff --git a/ac7d86e/internal/ui/types.go b/ac7d86e/internal/ui/types.go
deleted file mode 100644
index 6d03ad8..0000000
--- a/ac7d86e/internal/ui/types.go
+++ /dev/null
@@ -1,39 +0,0 @@
-type task struct {
-	id             int
-	summary        string
-	createdAt      time.Time
-	updatedAt      time.Time
-	trackingActive bool
-	secsSpent      int
-	active         bool
-	title          string
-	desc           string
-}
-type taskLogEntry struct {
-	id          int
-	taskId      int
-	taskSummary string
-	beginTs     time.Time
-	endTs       time.Time
-	secsSpent   int
-	comment     string
-	title       string
-	desc        string
-}
-type activeTaskDetails struct {
-	taskId              int
-	taskSummary         string
-	lastLogEntryBeginTs time.Time
-}
-type taskReportEntry struct {
-	taskId      int
-	taskSummary string
-	numEntries  int
-	secsSpent   int
-}
-func (t task) Title() string
-func (t task) Description() string
-func (t task) FilterValue() string
-func (e taskLogEntry) Title() string
-func (e taskLogEntry) Description() string
-func (e taskLogEntry) FilterValue() string
diff --git a/ac7d86e/internal/ui/ui.go b/53b6ad1/internal/ui/ui.go
index d9cad1a..670ef15 100644
--- a/ac7d86e/internal/ui/ui.go
+++ b/53b6ad1/internal/ui/ui.go
@@ -1 +1 @@
-func RenderUI(db *sql.DB)
+func RenderUI(db *sql.DB) error
diff --git a/ac7d86e/internal/ui/update.go b/53b6ad1/internal/ui/update.go
index 76cbabe..9047a01 100644
--- a/ac7d86e/internal/ui/update.go
+++ b/53b6ad1/internal/ui/update.go
@@ -1 +1 @@
-func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd)
+func (m Model) Update(msg tea.Msg) (tea.Model, tea.Cmd)
@@ -3 +3 @@ func (m recordsModel) Update(msg tea.Msg) (tea.Model, tea.Cmd)
-func (m model) shiftTime(direction timeShiftDirection, duration timeShiftDuration) error
+func (m Model) shiftTime(direction types.TimeShiftDirection, duration types.TimeShiftDuration) error
diff --git a/ac7d86e/internal/ui/utils.go b/53b6ad1/internal/ui/utils.go
index 632ba54..b5a7e92 100644
--- a/ac7d86e/internal/ui/utils.go
+++ b/53b6ad1/internal/ui/utils.go
@@ -6,3 +5,0 @@ type reportStyles struct {
-func RightPadTrim(s string, length int, dots bool) string
-func Trim(s string, length int) string
-func humanizeDuration(durationInSecs int) string
diff --git a/ac7d86e/internal/ui/view.go b/53b6ad1/internal/ui/view.go
index df21133..8adf0fd 100644
--- a/ac7d86e/internal/ui/view.go
+++ b/53b6ad1/internal/ui/view.go
@@ -1 +1 @@
-func (m model) View() string
+func (m Model) View() string
diff --git a/53b6ad1/internal/utils/utils.go b/53b6ad1/internal/utils/utils.go
new file mode 100644
index 0000000..ea99449
--- /dev/null
+++ b/53b6ad1/internal/utils/utils.go
@@ -0,0 +1,2 @@
+func RightPadTrim(s string, length int, dots bool) string
+func Trim(s string, length int) string
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
