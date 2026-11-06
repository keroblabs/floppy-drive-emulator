# source

STM32 firmware workspace placeholder.

Suggested structure:

- `src/` main firmware sources
- `inc/` headers
- `drivers/` HAL or LL wrappers
- `protocol/` Macintosh floppy protocol emulation layer

## Minimal firmware sketch (concept)

```c
// Pseudocode-only sketch
void app_main(void) {
    hw_init_gpio();
    hw_init_timers();
    hw_init_spi_flash();

    while (1) {
        sample_host_signals();
        service_emulated_track_data();
        handle_soft_eject_state_machine();
    }
}
```
