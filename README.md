# I2C EEPROM Driver

Block device driver for I2C based EEPROM devices such as the 24LC or AT24C series of devices.

EEPROM devices support much higher read/write cycles than flash based memory, at a cost in terms of size. Additionally, I2C EEPROM devices support byte-level read/writes with buffering of around 32bytes during writes.

**Note:** The I2C convention for EEPROM devices does not support inspection of the memory layout of the device, so the size must be specified during construction of the driver.

More info on EEPROM can be found on wikipedia:
https://en.wikipedia.org/wiki/EEPROM

## Basic example

``` cpp
// Here's an example using a M24M02-DRMN6 on GAP8
#include "mbed.h"
#include "I2CEEBlockDevice.h"

// Create EEPROM device on I2C bus with 32kbytes of memory, 256 byte page, 100 KHz
I2CEEBlockDevice i2cee(I2C0_SDA, I2C0_SCL, 0xA0, 256*1024, 256, 100000);

int main() {
    printf("i2cee test\n");

    // Initialize the device and print the memory layout
    i2cee.init();
    printf("i2cee size: %llu\n",         i2cee.size());
    printf("i2cee read size: %llu\n",    i2cee.get_read_size());
    printf("i2cee program size: %llu\n", i2cee.get_program_size());
    printf("i2cee erase size: %llu\n",   i2cee.get_erase_size());

    // Write "Hello World!" to the first block
    char *buffer = (char*)malloc(i2cee.get_erase_size());
    sprintf(buffer, "Hello World!\n");
    i2cee.erase(0, i2cee.get_erase_size());
    i2cee.program(buffer, 0, i2cee.get_erase_size());

    // Read back what was stored
    i2cee.read(buffer, 0, i2cee.get_erase_size());
    printf("%s", buffer);

    // Deinitialize the device
    i2cee.deinit();
    free(buffer);
}
```

## Saving struct example
``` cpp
// Here's an example using a M24M02-DRMN6 to store a C struct
#include "mbed.h"
#include "I2CEEBlockDevice.h"

#define BLOCK_SIZE 256

// Create EEPROM device on I2C bus with 32kbytes of memory
I2CEEBlockDevice i2cee(I2C0_SDA, I2C0_SCL, 0xA0, 256*1024, BLOCK_SIZE, 100000);

uint8_t setting_block_size;

struct t_setting {
    uint8_t version;
    char name[20];
} setting = {
    1,
    "Hello World!"
};

int main() {
    printf("i2cee struct test\n");

     // No. of bytes to be stored, but topped up to be multiplied by block size
    unsigned int setting_block_size = ceil(sizeof(setting)/(double)BLOCK_SIZE)*BLOCK_SIZE;

    // Temporary buffer
    char *buffer = (char*)malloc(setting_block_size);

    // Initialize the device
    i2cee.init();

    // Save struct to EEPROM
    printf("\nSaving struct version: %u, name: %s\n", setting.version, setting.name);
    memcpy(buffer, &setting, sizeof(setting));
    i2cee.program(buffer, 0, setting_block_size);

    // Get back what was stored
    t_setting tmp; //Re-make the struct
    memset(buffer, 0, sizeof(buffer)); // empty buffer, not nessesary but helps when debugging

    if (i2cee.read(buffer, 0, setting_block_size ) == 0){ // get data into buffer
        // Convert what we read into struct
        memcpy(&tmp, buffer, sizeof(tmp)); // copy only size of struct not setting_block_size
        printf("\nTemporary struct version: %u, name: %s\n", tmp.version, tmp.name);
    } else {
        printf("Error when reading\n");
    }

    // Deinitialize the device
    i2cee.deinit();
}
```

