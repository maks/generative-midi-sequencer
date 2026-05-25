cmake_minimum_required(VERSION 3.13)

# Pull in SDK (must be before project)
include(pico_sdk_import.cmake)

project(generative_midi_sequencer C CXX ASM)

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)

# Configure UART1 stdio for debug (GPIO 24 TX, GPIO 25 RX)
add_compile_definitions(
  PICO_DEFAULT_UART=1
  PICO_DEFAULT_UART_TX_PIN=24
  PICO_DEFAULT_UART_RX_PIN=25
)

# Initialize SDK
pico_sdk_init()

# Add src subdirectory
add_subdirectory(src)
