---
title: How to encrypt the NVS volume on the ESP32
layout: post
tags:
    - esp32
    - c
    - security
keywords:
    - esp32
    - c
    - security
---

There has been a lot of discussion around embedded device security during the last few years especially after [well-publicized DDoS incidents](https://blog.cloudflare.com/inside-mirai-the-infamous-iot-botnet-a-retrospective-analysis/) involving armies of hijacked IoT devices. The demand for higher levels of security has put pressure on manufacturers and software providers to adopt and support modern security protocols in order to mitigate the relevant risks especially given the widening spread of IoT devices.

An IoT device may embed various security-related artifacts in order to enable the use of relevant protocols, for example certificates for communicating securely with remote servers (mqtt, http etc.). It is really easy for a bad actor with physical access to the device to snatch binary images from the device's storage and search for such pieces of text. The possible leak of a security certificate may lead to escalated attacks into other parts of the infrastructure especially if the relevant permissions are incorrect or relaxed enough. In order to avoid that, we need to ensure that the device's storage is encrypted and that no sensitive information embedded in the firware image or elsewhere can be read even with physical access to the device.

In this article, we will demonstrate how to encrypt the non-volatile storage of the [ESP32](https://www.espressif.com/en/products/hardware/esp32/overview). The ESP32 is a very popular choice for building embedded solutions considering the SoC's capabilities (dual core, wifi, bluetooth), its low price (< 5$) and its [comprehensive SDK framework](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/) including a tcp/ip stack, http server/client (incl. TLS support), OTA firmware updates etc.

In general, encryption on the ESP32 is supported on the hardware level so as to prevent the recovery of (most) SPI flash contents using physical readouts. The ESP32 has two basic types of partitions: `app` which contain application-related artifacts such as the device firmware and `data` which contain arbitrary user data. The encryption of `app`-type partitions is fairly straight-forward and [well-documented](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/security/flash-encryption.html). The process roughly consists of building the firmware with support for encryption, flashing the device and leaving the rest to the bootloader. Partitions marked as `data` however are not handled automatically by the bootloader and require a different process with respect to encryption.

The [non-volatile storage of the ESP32](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/storage/nvs_flash.html) is a `data`-type partition that uses a portion of the underlying flash over SPI and is typically used for storing key-value pairs. These can include, for example, unique device identification strings, wifi configuration data (incl. passwords), device-specific security certificates etc., in other words stuff that we would like to keep private from curious eyes. The ESP32 supports NVS encryption but, as mentioned before, the process is a little bit more involved.

NVS supports two flavours of encryption:

* runtime-encryption whereby the application itself generates the key and encrypts/decrypts data on the fly, and
* build-time encryption whereby the nvs volume is pre-encrypted and flashed to the device

In the runtime-encryption method, the application generates a key using the [corresponding esp-idf function](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/storage/nvs_flash.html#_CPPv423nvs_flash_generate_keysPK15esp_partition_tP13nvs_sec_cfg_t) and uses this key in order to encrypt/decrypt data in the nvs volume at runtime. These can be, for example, wifi or other passwords that may be known only at runtime and not beforehand.

In the build-time encryption method, the NVS partition containing all the necessary key-value pairs is [prepared](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/storage/nvs_partition_gen.html) and encrypted for downloading to the device. This method can be used in cases where the NVS data are known at compile time -- example of such data include  unique device IDs, device-specific security certificates etc.

In both cases, a separate partition is necessary for storing the nvs encryption key. ESP-IDF provides a partition subtype for this purpose (type `data` and subtype `nvs_keys`) and handles its encryption transparently via the bootloader.

The use case that we'd like to demonstrate here is the baking of pre-existing data (unique device ID, security certificates) into the device at compile time. OK, so let's go through this procedure step-by-step by means of an example.

First, we need a custom partition table which can be configured using `idf.py menuconfig` and selecting "Custom Partition Table (CSV)" under the "Partition Table" menu (`CONFIG_PARTITION_TABLE_CUSTOM=y` and `CONFIG_PARTITION_TABLE_FILENAME="partitions.csv"`). Our `partitions.csv` file looks like this:

```csv
# ESP-IDF Partition Table
# Name,   Type, SubType, Offset,  Size, Flags
nvs,      data, nvs,       ,  0x4000,
otadata,  data, ota,       ,  0x2000,
phy_init, data, phy,       ,  0x1000,
factory,  0,    0,         , 1M,
ota_0,    0,    ota_0,    , 1M,
ota_1,    0,    ota_1,    , 1M,
nvs_key,  data, nvs_keys,         , 0x1000, encrypted
```

This partition table supports 3 app partitions (ESP's standard OTA scheme with 1 factory and 2 OTA partitions), one 16KB `nvs` partition and one 4KB `nvs_keys` partition. We have only specified partition sizes -- the offsets are calculated automatically by the tools. The `encrypted` flag of the `nvs_key` partition instructs the bootloader to automatically encrypt the contents, since we're going to be storing the nvs encryption keys there. (Of course it'd be great if we could do that for the `nvs` partition as well, but this feature is not supported unfortunately as the existence of this article demonstrates...)

Next, we are going to prepare our nvs image locally by specifying its contents using the CSV format as follows:

```csv
# NVS csv file
key,type,encoding,value
device_id,data,string,a_unique_value
cert,file,string,./path/to/certificate.pem.crt
pvkey,file,string,./path/to/private.pem.key
```

The actual image (`nvs.bin`) can be generated from the csv file (`nvs.csv`) using the partition generation tool (given a specified `$IDF_PATH`):

```bash
$ $IDF_PATH/components/nvs_flash/nvs_partition_generator/nvs_partition_gen.py generate nvs.csv nvs.bin 0x4000 // not encrypted
```

However, the resulting image `nvs.bin` will be unencrypted. If instead of the `generate` command we use the `encrypt` command, the resulting image will be encrypted and the tool will output a second image (`nvs_keys.bin`) with the contents of the encryption key:

```bash
$ $IDF_PATH/components/nvs_flash/nvs_partition_generator/nvs_partition_gen.py encrypt nvs.csv encrypted_nvs.bin 0x4000 --keygen --keyfile nvs_keys.bin
```

These two images can now be flashed to the device using `esptool.py` (part of the esp-idf distribution):

```bash
$ esptool.py -p PORT --before default_reset --after no_reset write_flash 0xa000 encrypted_nvs.bin
```

where `PORT` is the serial comm device address (something like `/dev/cu.usbserial-0001`) and `encrypted_nvs.bin` is the image file that we generated in our previous step. The flash location address `0xa000` can be discovered either by inspecting the esp32 serial output (where the partitions are printed out) or by using the `gen_esp32part.py` utility on the project's partition image (e.g. `gen_esp32part.py build/partition_table/partition-table.bin`).

The `nvs_keys` image also needs to be downloaded to the device:

```bash
$ esptool.py -p PORT --before default_reset --after no_reset write_flash 0x320000 nvs_keys.bin
```

We're almost done. On the application side, we now need to initialize the secure nvs volume like so:

```c
esp_err_t nvs_secure_initialize() {
    static const char *nvs_tag = "nvs";
    esp_err_t err = ESP_OK;

    // 1. find partition with nvs_keys
    const esp_partition_t *partition = esp_partition_find_first(ESP_PARTITION_TYPE_DATA,
                                                                ESP_PARTITION_SUBTYPE_DATA_NVS_KEYS,
                                                                "nvs_key");
    if (partition == NULL) {
        ESP_LOGE(nvs_tag, "Could not locate nvs_key partition. Aborting.");
        return ESP_FAIL;
    }

    // 2. read nvs_keys from key partition
    nvs_sec_cfg_t cfg;
    if (ESP_OK != (err = nvs_flash_read_security_cfg(partition, &cfg))) {
        ESP_LOGE(nvs_tag, "Failed to read nvs keys (rc=0x%x)", err);
        return err;
    }

    // 3. initialize nvs partition
    if (ESP_OK != (err = nvs_flash_secure_init(&cfg))) {
        ESP_LOGE(nvs_tag, "failed to initialize nvs partition (err=0x%x). Aborting.", err);
        return err;
    };

    return err;
}

void app_main() {
    esp_err_t err = nvs_secure_initialize();
    if (err != ESP_OK) {
        ESP_LOGE("main", "Failed to initialize nvs (rc=0x%x). Halting.", err);
        while(1) { vTaskDelay(100); }
    }

    // rest of application code goes here
    // ...
}
```

Once the app is built and flashed (using `idf.py encrypted-flash`), we're good to go with our encrypted NVS volume and our IoT device can now be safely deployed to the field.
