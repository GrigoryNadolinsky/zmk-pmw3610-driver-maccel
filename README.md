PMW3610 driver implementation for ZMK with at least Zephyr 3.5

This work is based on [ufan's implementation](https://github.com/ufan/zmk/tree/support-trackpad) of the driver.

## Installation

Only GitHub actions builds are covered here. Local builds are different for each user, therefore it's not possible to cover all cases.

Include this project on your ZMK's west manifest in `config/west.yml`:

```yml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/petejohanson
    - name: inorichi
      url-base: https://github.com/inorichi
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: feat/pointers-move-scroll
      import: app/west.yml
    - name: zmk-pmw3610-driver
      remote: inorichi
      revision: main
  self:
    path: config
```

Then, edit your `build.yml` to look like this, 3.5 is now on main:

```yml
on: [workflow_dispatch]

jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@main
```

Now, update your `board.overlay` adding the necessary bits (update the pins for your board accordingly):

```dts
&pinctrl {
    spi0_default: spi0_default {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 8)>,
                <NRF_PSEL(SPIM_MOSI, 0, 17)>,
                <NRF_PSEL(SPIM_MISO, 0, 17)>;
        };
    };

    spi0_sleep: spi0_sleep {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 8)>,
                <NRF_PSEL(SPIM_MOSI, 0, 17)>,
                <NRF_PSEL(SPIM_MISO, 0, 17)>;
            low-power-enable;
        };
    };
};

&spi0 {
    status = "okay";
    compatible = "nordic,nrf-spim";
    pinctrl-0 = <&spi0_default>;
    pinctrl-1 = <&spi0_sleep>;
    pinctrl-names = "default", "sleep";
    cs-gpios = <&gpio0 20 GPIO_ACTIVE_LOW>;

    trackball: trackball@0 {
        status = "okay";
        compatible = "pixart,pmw3610";
        reg = <0>;
        spi-max-frequency = <2000000>;
        irq-gpios = <&gpio0 6 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;

        /*   optional features   */
        // snipe-layers = <1>;
        // scroll-layers = <2 3>;
        // automouse-layer = <4>;
    };
};

/ {
  trackball_listener {
    compatible = "zmk,input-listener";
    device = <&trackball>;

  };
};
```

Now enable the driver config in your `board.config` file (read the Kconfig file to find out all possible options):

```conf
CONFIG_SPI=y
CONFIG_INPUT=y
CONFIG_ZMK_MOUSE=y
CONFIG_PMW3610=y
```


## Mouse Acceleration
This module also provides driver level mouse acceleration ported from [QMK maccel, developed by Wimads and finrod09](https://github.com/finrod09/qmk_userspace_features/tree/main/maccel). 
To enable it, first make sure to disable OS level mouse acceleration and then add this to your `board.config`.

```conf
CONFIG_PMW3610_MACCEL=y
CONFIG_PMW3610_CPI_DIVIDOR=1
```

All the parameters for the maccel equation can be changed(e.g. GROWTH_RATE,TAKE_OFF), to learn more about how these values 
affect the acceleration read [original documentation](https://github.com/finrod09/qmk_userspace_features/blob/main/maccel/readme.md#configuration). To change the parameters for maccell, you should multiply the value you intend to use by 100, as in the following example.

```conf
# 200 instead of 2.0 for qmk
CONFIG_PMW3610_MACCEL_TAKEOFF=200
# 025 instead of 0.25 for qmk
CONFIG_PMW3610_MACCEL_GROWTH_RATE=025
# 220 instead of 2.2 for qmk
CONFIG_PMW3610_MACCEL_OFFSET=220
# 020 instead of 0.2 for qmk
CONFIG_PMW3610_MACCEL_LIMIT=020
```
