# evcc for Catan C1 (PLCnext)

[evcc](https://github.com/evcc-io/evcc) app image for **Phoenix Contact Catan C1**

> [!WARNING]
> This is an unofficial community project. It is not affiliated with, endorsed by, or supported by Phoenix Contact.
> No support is provided. Use at your own risk; the software comes without any warranty (see [LICENSE](LICENSE)).

- Firm Code `0000000`
- Product Codes `1337002`

## Supported Devices

- Controller - Catan C1 - 1886438 (Phoenix Contact)

## WBM

![plcNextControl](/docs/img/plcNextControl.png)

## Known issues

- No support for ch340/ch341 chipsets

## Dev

### Build

```bash
sudo apt install -y squashfs-tools docker-ce docker-ce-cli
bash build.sh <app version>   # e.g. bash build.sh 1.2.0
```

### Release

Publishing a GitHub release (tag e.g. `v1.2.0`) builds the app with that version and attaches a `.zip` with the `.app`, `.md5` and `LICENSE` files to the release.
