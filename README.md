# Keygen Go SDK [![godoc reference](https://godoc.org/github.com/keygen-sh/keygen-go?status.png)](https://godoc.org/github.com/keygen-sh/keygen-go)

Package [`keygen`](https://pkg.go.dev/github.com/keygen-sh/keygen-go) allows Go programs to
license and remotely update themselves using the [keygen.sh](https://keygen.sh) service.

## Config

### `keygen.Account`

`Account` is your Keygen account ID used globally in the binding. All requests will be made
to this account. This should be hard-coded into your app.

```go
keygen.Account = "1fddcec8-8dd3-4d8d-9b16-215cac0f9b52"
```

### `keygen.Product`

`Product` is your Keygen product ID used globally in the binding. All license validations and
upgrade requests will be scoped to this product. This should be hard-coded into your app.

```go
keygen.Product = "1f086ec9-a943-46ea-9da4-e62c2180c2f4"
```

### `keygen.Token`

`Token` is an activation token belonging to the licensee. This will be used for license validations,
activations, deactivations and upgrade requests. You will need to prompt the end-user for this.

```go
keygen.Token = "activ-d66e044ddd7dcc4169ca9492888435d3v3"
```

### `keygen.PublicKey`

`PublicKey` is your Keygen account's hex-encoded Ed25519 public key, used for verifying signed license keys
and API response signatures. When set, API response signatures will automatically be verified. You may
leave it blank to skip verifying response signatures. This should be hard-coded into your app.

```go
keygen.PublicKey = "e8601e48b69383ba520245fd07971e983d06d22c4257cfd82304601479cee788"
```

### `keygen.Channel`

`Channel` is the release channel used when checking for upgrades. Defaults to `stable`. You may
provide a different channel, one of: `stable`, `rc`, `beta`, `alpha` or `dev`.

```go
keygen.Channel = "dev"
```

### `keygen.Platform`

`Platform` is the release platform used when checking for upgrades and when activating machines.
Defaults to `runtime.GOOS + "_" + runtime.GOARCH`. You may provide a custom platform.

```go
keygen.Platform = "win32"
```

### `keygen.UpgradeKey`

`UpgradeKey` is your personal hex-encoded Ed25519ph public key, used for verifying that an upgrade was
signed by yourself or your team using your private publishing key. This should not be equal to
`keygen.PublicKey` — they are different keys for different purposes. When set, a release's signature
will be verified before an upgrade is installed. This should be hard-coded into your app.

You can generate a publishing key pair using Keygen's CLI.

```go
keygen.UpgradeKey = "5ec69b78d4b5d4b624699cef5faf3347dc4b06bb807ed4a2c6740129f1db7159"
```

### `keygen.Logger`

`Logger` is a leveled logger implementation used for printing debug, informational, warning, and
error messages. The default log level is `LogLevelError`. You may provide your own logger which
implements `LoggerInterface`.

```go
keygen.Logger = &CustomLogger{Level: LogLevelDebug}
```

## Usage

### `keygen.Validate(fingerprint)`

To validate a license, configure `keygen.Account` and `keygen.Product` with your Keygen account
details. Then prompt the user for their license token and set `keygen.Token`.

The `Validate` method accepts zero or more fingerprints, which can be used to scope a license
validation to a particular fingerprint. It will return a `License` object as well as any
validation errors that occur. The `License` object can be used to perform additional actions,
such as `license.Activate(fingerprint)`.

```go
license, err := keygen.Validate(fingerprint)
switch {
case err == keygen.ErrLicenseNotActivated:
  fmt.Println("License is not activated!")

  return
case err == keygen.ErrLicenseExpired:
  fmt.Println("License is expired!")

  return
case err != nil:
  fmt.Println("License is invalid!")

  return
}

fmt.Println("License is valid!")
```

### `keygen.Upgrade(currentVersion)`

Check for an upgrade. When an upgrade is available, a `Release` will be returned which will
allow the update to be installed, replacing the currently running binary. When an upgrade
is not available, an `ErrUpgradeNotAvailable` error will be returned indicating the current
version is up-to-date.

```go
release, err := keygen.Upgrade(currentVersion)
switch {
case err == keygen.ErrUpgradeNotAvailable:
  fmt.Println("No upgrade available, already at the latest version!")

  return
case err != nil:
  fmt.Println("Upgrade check failed!")

  return
}

if err := release.Install(); err != nil {
  fmt.Println("Upgrade install failed!")

  return
}

fmt.Println("Upgrade complete! Please restart.")
```

### `keygen.Genuine(licenseKey, schemeCode)`

Cryptographically verify and decode a signed license key. This is useful for checking if a license
key is genuine in offline or air-gapped environments. Returns the key's decoded dataset and any
errors that occurred during cryptographic verification, e.g. `ErrLicenseNotGenuine`.

Requires that `keygen.PublicKey` is set.

```go
dataset, err := keygen.Genuine(licenseKey, keygen.SchemeCodeEd25519)
switch {
case err == keygen.ErrLicenseNotGenuine:
  fmt.Println("License key is not genuine!")

  return
case err != nil:
  fmt.Println("Genuine check failed!")

  return
}

fmt.Println("License is genuine!")
fmt.Printf("Decoded dataset: %s\n", dataset)
```

---

## Examples

### License activation example

```go
package main

import "github.com/keygen-sh/keygen-go"

func main() {
  keygen.Account = os.Getenv("KEYGEN_ACCOUNT")
  keygen.Product = os.Getenv("KEYGEN_PRODUCT")
  keygen.Token = os.Getenv("KEYGEN_TOKEN")

  // The current device's fingerprint (could be e.g. MAC, mobo ID, GUID, etc.)
  fingerprint := uuid.New().String()

  // Validate the license for the current fingerprint
  license, err := keygen.Validate(fingerprint)
  switch {
  case err == keygen.ErrLicenseNotActivated:
    // Activate the current fingerprint
    machine, err := license.Activate(fingerprint)
    switch {
    case err == keygen.ErrMachineLimitExceeded:
      fmt.Println("Machine limit has been exceeded!")

      return
    case err != nil:
      fmt.Println("Machine activation failed!")

      return
    }
  case err == keygen.ErrLicenseExpired:
    fmt.Println("License is expired!")

    return
  case err != nil:
    fmt.Println("License is invalid!")

    return
  }

  fmt.Println("License is activated!")
}
```

### Automatic upgrade example

```go
package main

import "github.com/keygen-sh/keygen-go"

// The current version of the program
const currentVersion = "1.0.0"

func main() {
  keygen.Account = os.Getenv("KEYGEN_ACCOUNT")
  keygen.Product = os.Getenv("KEYGEN_PRODUCT")
  keygen.Token = os.Getenv("KEYGEN_TOKEN")

  fmt.Printf("Current version: %s\n", currentVersion)
  fmt.Println("Checking for upgrades...")

  // Check for upgrade
  release, err := keygen.Upgrade(currentVersion)
  switch {
  case err == keygen.ErrUpgradeNotAvailable:
    fmt.Println("No upgrade available, already at the latest version!")

    return
  case err != nil:
    fmt.Println("Upgrade check failed!")

    return
  }

  // Download the upgrade and install it
  err = release.Install()
  if err != nil {
    fmt.Println("Upgrade install failed!")

    return
  }

  fmt.Printf("Upgrade complete! Installed version: %s\n", release.Version)
  fmt.Println("Restart to finish installation...")
}
```

### Machine heartbeats example

```go
package main

import "github.com/keygen-sh/keygen-go"

func main() {
  keygen.Account = os.Getenv("KEYGEN_ACCOUNT")
  keygen.Product = os.Getenv("KEYGEN_PRODUCT")
  keygen.Token = os.Getenv("KEYGEN_TOKEN")

  // The current device's fingerprint (could be e.g. MAC, mobo ID, GUID, etc.)
  fingerprint := uuid.New().String()

  // Keep our example process alive
  done := make(chan bool, 1)

  // Validate the license for the current fingerprint
  license, err := keygen.Validate(fingerprint)
  switch {
  case err == keygen.ErrLicenseNotActivated:
    // Activate the current fingerprint
    machine, err := license.Activate(fingerprint)
    if err != nil {
      fmt.Println("Machine activation failed!")

      panic(err)
    }

    // Handle SIGINT and gracefully deactivate the machine
    sigs := make(chan os.Signal, 1)

    signal.Notify(sigs, os.Interrupt)

    go func() {
      for sig := range sigs {
        fmt.Printf("Caught %v, deactivating machine and gracefully exiting...\n", sig)

        if err := machine.Deactivate(); err != nil {
          fmt.Println("Machine deactivation failed!")

          panic(err)
        }

        fmt.Println("Machine was deactivated!")
        fmt.Println("Exiting...")

        done <- true
      }
    }()

    // Start a heartbeat monitor for the current machine
    errs := machine.Monitor()

    go func() {
      err := <-errs

      // We want to kill the current process if our heartbeat ping fails
      panic(err)
    }()

    fmt.Println("Machine is activated and monitored!")
  case err != nil:
    fmt.Println("License is invalid!")

    panic(err)
  }

  fmt.Println("License is valid!")

  <-done
}
```
