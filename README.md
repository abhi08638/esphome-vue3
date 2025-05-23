For issues, please go to [the discussion board](https://github.com/emporia-vue-local/esphome/discussions).

## READ OVER ALL INSTALL INSTRUCTIONS PRIOR TO FLASHING
https://www.emporiaenergy.com/wp-content/uploads/2024/03/Emporia-Vue-Gen-3-Installation-Guide-v1.20.pdf

## Flashing Emporia Vue 3 to ESPHome
https://github.com/emporia-vue-local/esphome/discussions/264#discussioncomment-12517829

## 3D printable Flashing Jig
https://www.printables.com/model/1152988-emporia-vue-v3-esp-home-flash-jig-w-fixed-toleranc/comments

## Writing configuration
My setup is for a US home using single split-phase power with a tesla solar array tied to a powerwall+ that feeds power to my main panel via a breaker (clamps 1 & 2).

NOTE: It is NOT possible to see the energy going to/from the powerwall in my setup. The "Tesla Power" sensor will simply give you the net output of solar energy + powerwall energy. 

Here's a starting point for a configuration, save it to `<yourfilename>.yaml` into project folder:

```yaml
esphome:
  name: emporiavue3

substitutions:
  display_name: EmpVu3
  
  circuit_01_02: "Tesla"
  circuit_03: "BRK 13"
  circuit_04: "BRK 11"
  circuit_05: "BRK 9"
  circuit_06: "BRK 7"
  circuit_07: "BRK 5"
  circuit_08: "BRK 2"
  circuit_09: "BRK 4"
  circuit_10: "BRK 6"
  circuit_11: "BRK 8"
  circuit_12_13: "EV Charger"
  circuit_14_15: "Heat Pump"
  
  # Icons for Home Assistant
  icon_circuit: "mdi:power-plug"
  # icon_current: "mdi:current-ac"
  icon_meter:   "mdi:meter-electric"
  icon_pwr:   "mdi:transmission-tower-import"

external_components:
#  - source: github://emporia-vue-local/esphome@dev
# IMPORTANT: digiblur's repo has a fix for Phase A power not reporting correctly since the pins were changed in later hardware revisions. 
  - source: github://digiblur/esphome-vue3@dev
    components:
      - emporia_vue
      
esp32:
  board: esp32dev
  framework:
    type: esp-idf
    version: recommended

# https://github.com/emporia-vue-local/esphome/discussions/264?sort=old#discussioncomment-8894510
# ethernet:
#   type: RTL8201
#   mdc_pin: GPIO32
#   mdio_pin: GPIO33
#   clk_mode: GPIO0_IN

preferences:
  # the default of 1min is far too short--flash chip is rated
  # for approx 100k writes.
  flash_write_interval: "48h"   

# Enable Home Assistant API
api:
  encryption:
    key: "KEY"

ota:
  platform: esphome
  password: "key"
  on_error:
    then:
      - button.press: safe_mode_button

logger:
  logs:
    sensor: INFO

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  on_connect:
    - light.turn_on: wifi_led
  on_disconnect:
    - light.turn_off: wifi_led
  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Emporia-Vue Fallback Hotspot"
    password: "password"

captive_portal:

i2c:
  sda:
    number: 5
    ignore_strapping_warning: true
  scl: 18
  scan: false
  #frequency: 150kHz  # recommended range is 50-200kHz but does not work after esphome 2024.10
  frequency: 400kHz  # IMPORTANT: Handle Failed to read from sensor due to I2C error 3
  timeout: 1ms # IMPORTANT: Handle Failed to read from sensor due to I2C error 3
  id: i2c_a

button:
  # https://github.com/emporia-vue-local/esphome/discussions/264?sort=old#discussioncomment-11621009
  - platform: restart
    name: Restart
    id: restart_button
    icon: mdi:restart

  - platform: safe_mode
    name: Restart (Safe Mode)
    id: safe_mode_button
    icon: mdi:restart-alert

time:
  # - platform: sntp
  #   id: sntp_time
  # https://esphome.io/components/time/homeassistant
  - platform: homeassistant    

# https://github.com/emporia-vue-local/esphome/discussions/264?sort=old#discussioncomment-9788390
# Configure the two status LEDs on the Emporia Vue v3 case.
light:
  - platform: status_led
    id: wifi_led
    name: "WiFi LED"
    pin:
      number: 2
      ignore_strapping_warning: true
    entity_category: diagnostic
    restore_mode: ALWAYS_ON

  - platform: status_led
    id: ethernet_led
    name: "Ethernet LED"
    pin: 4
    entity_category: config
    restore_mode: ALWAYS_OFF

# these are called references in YAML. They allow you to reuse
# this configuration in each sensor, while only defining it once

# Using a throttle_average filter let's us specify a time to average over compared to the sliding window average that depends on knowing the update frequency, so it doesn't matter which revision Emporia Vue you have
# The raw power measurements are left as internal sensors with no throttle
# The total daily energy sensors use these internal sensors, and should accurately capture very fast changes in the power measurements
# Apply a time based throttle filter to the energy sensors so as to not publish updates too frequently; this sends the most recent value after the specified time has passed since last publishing
# Copy sensors publish the raw internal sensors and filter them with throttle_average
# The template total power and balance power sensors use the raw internal sensors as their source; this should improve accuracy as we are not averaging averages
# throttle_average filter is applied after combining the internal sensors to reduce publishing rates
# Adam Jaques wrote a great blog post that motivated using these filters with copy sensors to improve accuracy:
# https://www.technowizardry.net/2023/02/local-energy-monitoring-using-the-emporia-vue-2/

# The template total and balance power sensors now have an update_interval of never, and they are instead updated only via this trigger
# This let's us avoid having to specify a time based update_interval that relies on the hardware revision's internal update rate
# Having more accurate power sensors that update every time the Emporia hardware sends new measurements should improve the daily energy sensor's accuracy for total power and balance power
# This behaves much like the built in ESPHome automation trigger on_value. The on_update trigger is guaranteed to execute once after every individual sensor in the component has updated. This allows the template sensors to simultaneously update after all the internal power sensors have finished publishing new values. In contrast, the on_value trigger is executed after a specific individual sensor has updated.

.defaultfilters:
  - &throttle_avg
    # average all raw readings together over a 5 second span before publishing
    throttle_average: 10s
  - &throttle_time
    # only send the most recent measurement every 60 seconds
    throttle: 60s
  - &moving_avg
    # we capture a new sample every 0.24 seconds, so the time can
    # be calculated from the number of samples as n * 0.24.
    sliding_window_moving_average:
      # we average over the past 5.74 seconds (24 * 0.24)
      window_size: 24
      # we push a new value every 2.88 seconds (12 * 0.24)
      send_every: 12
  - &invert
    # invert and filter out any values below 0.
    lambda: 'return max(-x, 0.0f);'
  - &pos
    # filter out any values below 0.
    lambda: 'return max(x, 0.0f);'
  - &abs
    # take the absolute value of the value
    lambda: 'return abs(x);'

# https://gist.github.com/bradsjm/c2562df0e891f26e27191a31d617d471
text_sensor:
  - platform: wifi_info
    ip_address:
      name: "IP Address"
    ssid:
      name: "Connected SSID"
    mac_address:
      name: "MAC Address"

sensor:
  - platform: emporia_vue
    i2c_id: i2c_a
    phases:
      - id: phase_a  # Verify that this specific phase/leg is connected to correct input wire color on device listed below
        input: BLACK  # Vue device wire color
        calibration: 0.0192  # 0.022 is used as the default as starting point but may need adjusted to ensure accuracy
        # To calculate new calibration value use the formula <in-use calibration value> * <accurate voltage> / <reporting voltage>
        # I used the voltage reported from the tesla powerwall integration as <accurate voltage>
        voltage:
          #name: "Phase A Voltage"
          id: phase_a_voltage
          filters: [*throttle_time, *pos]
      - id: phase_b  # Verify that this specific phase/leg is connected to correct input wire color on device listed below
        input: RED  # Vue device wire color
        calibration: 0.0192  # 0.022 is used as the default as starting point but may need adjusted to ensure accuracy
        # To calculate new calibration value use the formula <in-use calibration value> * <accurate voltage> / <reporting voltage>
        # I used the voltage reported from the tesla powerwall integration as <accurate voltage>
        voltage:
          #name: "Phase B Voltage"
          id: phase_b_voltage
          filters: [*throttle_time, *pos]

    ct_clamps:
      # Do not specify a name for any of the power sensors here, only an id. This leaves the power sensors internal to ESPHome.
      # Copy sensors will filter and then send power measurements to HA
      # These non-throttled power sensors are used for accurately calculating energy
      # See https://github.com/emporia-vue-local/esphome/pull/200
      
      # Power pulled from grid
      - phase_id: phase_a
        input: "A"  # Verify the CT going to this device input also matches the phase/leg
        power:
          id: phase_a_pwr
          filters: [*pos] # This measures energy pulled from grid on phase A
      - phase_id: phase_b
        input: "B"  # Verify the CT going to this device input also matches the phase/leg
        power:
          id: phase_b_pwr
          filters: [*pos] # This measures energy pulled from grid on phase B
      
      #Power pushed to grid
      - phase_id: phase_a
        input: "A"  # Verify the CT going to this device input also matches the phase/leg
        power:
          id: phase_a_pwr_return
          filters: [*invert]  # This measures energy pushed to grid on phase A
      - phase_id: phase_b
        input: "B"  # Verify the CT going to this device input also matches the phase/leg
        power:
          id: phase_b_pwr_return
          filters: [*invert]  # This measures energy pushed to grid on phase B
      
      # Pay close attention to set the phase_id for each breaker by matching it to the phase/leg it connects to in the panel
      - { phase_id: phase_a, input:  "1", power: { id:  c1_pwr } }
      - { phase_id: phase_b, input:  "2", power: { id:  c2_pwr } }
      - { phase_id: phase_b, input:  "3", power: { id:  c3_pwr, filters: [*pos] } }
      - { phase_id: phase_a, input:  "4", power: { id:  c4_pwr, filters: [*pos] } }
      - { phase_id: phase_b, input:  "5", power: { id:  c5_pwr, filters: [*pos] } }
      - { phase_id: phase_a, input:  "6", power: { id:  c6_pwr, filters: [*pos] } }
      - { phase_id: phase_b, input:  "7", power: { id:  c7_pwr, filters: [*pos] } }
      - { phase_id: phase_b, input:  "8", power: { id:  c8_pwr, filters: [*pos] } }
      - { phase_id: phase_a, input:  "9", power: { id:  c9_pwr, filters: [*pos] } }
      - { phase_id: phase_b, input: "10", power: { id:  c10_pwr, filters: [*pos] } }
      - { phase_id: phase_a, input: "11", power: { id:  c11_pwr, filters: [*pos] } }
      - { phase_id: phase_b, input: "12", power: { id:  c12_pwr, filters: [*pos] } }
      - { phase_id: phase_a, input: "13", power: { id:  c13_pwr, filters: [*pos] } }
      - { phase_id: phase_b, input: "14", power: { id:  c14_pwr, filters: [*pos] } }
      - { phase_id: phase_a, input: "15", power: { id:  c15_pwr, filters: [*pos] } }
    on_update:
      then:
        - component.update: total_grid_pwr
        - component.update: total_grid_pwr_return_helper
        - component.update: total_grid_pwr_return
        - component.update: total_home_pwr
        - component.update: circuit_01_02_pwr
        - component.update: circuit_12_13_pwr
        - component.update: circuit_14_15_pwr
  
  # Total pulled from grid
  - platform: template
    #name: "Total Grid Power"
    lambda: return (id(phase_a_pwr).state + id(phase_b_pwr).state)-(id(phase_a_pwr_return).state + id(phase_b_pwr_return).state);
    update_interval: never # will be updated after all power sensors update via on_update trigger
    id: total_grid_pwr
    device_class: power
    state_class: measurement
    unit_of_measurement: "W"
    filters: *pos
  - platform: total_daily_energy
    name: "Total Grid Daily Energy"
    power_id: total_grid_pwr
    accuracy_decimals: 0
    restore: false
    filters: *throttle_time
    state_class: total_increasing
    device_class: energy

# Total pushed to grid
  - platform: template
    #name: "Total Grid Power Return Helper"
    lambda: return (id(phase_a_pwr_return).state + id(phase_b_pwr_return).state);
    update_interval: never # will be updated after all power sensors update via on_update trigger
    id: total_grid_pwr_return_helper
    device_class: power
    state_class: measurement
    unit_of_measurement: "W"
    #filters: *throttle_avg
  - platform: template 
    lambda:
      if(((id(phase_a_pwr).state+id(phase_b_pwr).state)-id(total_grid_pwr_return_helper).state)<-10){return ((id(phase_a_pwr).state+id(phase_b_pwr).state)-id(total_grid_pwr_return_helper).state);}
      else {return 0;}
    update_interval: never
    id: total_grid_pwr_return
    device_class: power
    state_class: measurement
    unit_of_measurement: "W"
    filters: *abs
  - platform: total_daily_energy
    name: "Total Grid Daily Energy Return"
    power_id: total_grid_pwr_return
    accuracy_decimals: 0
    restore: false
    filters: *throttle_time
    state_class: total_increasing
    device_class: energy
    
# Total home usage
  - platform: template
    #name: "Total Home Power"
    lambda: 
      int total = id(c3_pwr).state + id(c4_pwr).state + id(c5_pwr).state + id(c6_pwr).state + id(c7_pwr).state + id(c8_pwr).state + id(c9_pwr).state+ id(c10_pwr).state + id(c11_pwr).state + id(circuit_14_15_pwr).state + id(circuit_12_13_pwr).state;
      int tesla_pwr = id(circuit_01_02_pwr).state;
      if(tesla_pwr>1){return (total+tesla_pwr);}
      else {return total;}
    update_interval:  never   # will be updated after all power sensors update via on_update trigger
    id: total_home_pwr
    device_class: power
    state_class: measurement
    unit_of_measurement: "W"
    #filters: *throttle_avg
  - platform: total_daily_energy
    name: "Total Home Daily Energy"
    power_id: total_home_pwr
    accuracy_decimals: 0
    restore: false
    filters: *throttle_time
    state_class: total_increasing
    device_class: energy

# Power from Tesla Solar and Powerwall
  - platform: template
    #name: "${circuit_01_02} Power"
    lambda: return (id(c1_pwr).state + id(c2_pwr).state);
    update_interval: never # will be updated after all power sensors update via on_update trigger
    id: circuit_01_02_pwr
    device_class: power
    state_class: measurement
    unit_of_measurement: "W"
    #filters: *throttle_avg
  - platform: total_daily_energy
    name: "${circuit_01_02} Daily Energy"
    power_id: circuit_01_02_pwr
    accuracy_decimals: 0
    restore: false
    filters: *throttle_time
   # state_class: total_increasing
    device_class: energy

# 12 13 Power
  - platform: template
    #name: "${circuit_12_13} Power"
    lambda: return id(c12_pwr).state + id(c13_pwr).state;
    update_interval: never # will be updated after all power sensors update via on_update trigger
    id: circuit_12_13_pwr
    device_class: power
    state_class: measurement
    unit_of_measurement: "W"
    #filters: *throttle_avg
  - platform: total_daily_energy
    name: "${circuit_12_13} Daily Energy"
    power_id: circuit_12_13_pwr
    accuracy_decimals: 0
    restore: false
    filters: *throttle_time
    state_class: total_increasing
    device_class: energy
    
# 14 15 Power
  - platform: template
    #name: "${circuit_14_15} Power"
    lambda: return id(c14_pwr).state + id(c15_pwr).state;
    update_interval: never # will be updated after all power sensors update via on_update trigger
    id: circuit_14_15_pwr
    device_class: power
    state_class: measurement
    unit_of_measurement: "W"
    #filters: *throttle_avg
  - platform: total_daily_energy
    name: "${circuit_14_15} Daily Energy"
    power_id: circuit_14_15_pwr
    accuracy_decimals: 0
    restore: false
    filters: *throttle_time
    state_class: total_increasing
    device_class: energy
  
# Copy sensors filter and send the power state to HA
  - { platform: copy, name:                    "Total Grid Power", source_id:                   total_grid_pwr, filters: *throttle_avg, icon:   $icon_pwr }
  - { platform: copy, name:             "Total Grid Power Return", source_id:            total_grid_pwr_return, filters: *throttle_avg, icon:   $icon_pwr }
  - { platform: copy, name:                    "Total Home Power", source_id:                   total_home_pwr, filters: *throttle_avg, icon:   $icon_pwr }
  - { platform: copy, name:              "${circuit_01_02} Power", source_id:                circuit_01_02_pwr, filters: *throttle_avg, icon: $icon_circuit }
  - { platform: copy, name:                 "${circuit_03} Power", source_id:                           c3_pwr, filters: *throttle_avg, icon: $icon_circuit }
  - { platform: copy, name:                 "${circuit_04} Power", source_id:                           c4_pwr, filters: *throttle_avg, icon: $icon_circuit }
  - { platform: copy, name:                 "${circuit_05} Power", source_id:      		                  c5_pwr, filters: *throttle_avg, icon: $icon_circuit }
  - { platform: copy, name:                 "${circuit_06} Power", source_id:           	              c6_pwr, filters: *throttle_avg, icon: $icon_circuit }
  - { platform: copy, name:                 "${circuit_07} Power", source_id:                	          c7_pwr, filters: *throttle_avg, icon: $icon_circuit }
  - { platform: copy, name:                 "${circuit_08} Power", source_id:                           c8_pwr, filters: *throttle_avg, icon: $icon_circuit }
  - { platform: copy, name:                 "${circuit_09} Power", source_id:                           c9_pwr, filters: *throttle_avg, icon: $icon_circuit }
  - { platform: copy, name:                 "${circuit_10} Power", source_id:                          c10_pwr, filters: *throttle_avg, icon: $icon_circuit }
  - { platform: copy, name:                 "${circuit_11} Power", source_id:                          c11_pwr, filters: *throttle_avg, icon: $icon_circuit }
  - { platform: copy, name:              "${circuit_12_13} Power", source_id:                circuit_12_13_pwr, filters: *throttle_avg, icon: $icon_circuit }
  - { platform: copy, name:              "${circuit_14_15} Power", source_id:                circuit_14_15_pwr, filters: *throttle_avg, icon: $icon_circuit }

  # All breakers total daily energy 
  - { power_id:  c3_pwr, platform: total_daily_energy, accuracy_decimals: 0, restore: false, name: "${circuit_03} Daily Energy", filters: *throttle_time, icon: $icon_meter }
  - { power_id:  c4_pwr, platform: total_daily_energy, accuracy_decimals: 0, restore: false, name: "${circuit_04} Daily Energy", filters: *throttle_time, icon: $icon_meter }
  - { power_id:  c5_pwr, platform: total_daily_energy, accuracy_decimals: 0, restore: false, name: "${circuit_05} Daily Energy", filters: *throttle_time, icon: $icon_meter }
  - { power_id:  c6_pwr, platform: total_daily_energy, accuracy_decimals: 0, restore: false, name: "${circuit_06} Daily Energy", filters: *throttle_time, icon: $icon_meter }
  - { power_id:  c7_pwr, platform: total_daily_energy, accuracy_decimals: 0, restore: false, name: "${circuit_07} Daily Energy", filters: *throttle_time, icon: $icon_meter }
  - { power_id:  c8_pwr, platform: total_daily_energy, accuracy_decimals: 0, restore: false, name: "${circuit_08} Daily Energy", filters: *throttle_time, icon: $icon_meter }
  - { power_id:  c9_pwr, platform: total_daily_energy, accuracy_decimals: 0, restore: false, name: "${circuit_09} Daily Energy", filters: *throttle_time, icon: $icon_meter }
  - { power_id: c10_pwr, platform: total_daily_energy, accuracy_decimals: 0, restore: false, name: "${circuit_10} Daily Energy", filters: *throttle_time, icon: $icon_meter }
  - { power_id: c11_pwr, platform: total_daily_energy, accuracy_decimals: 0, restore: false, name: "${circuit_11} Daily Energy", filters: *throttle_time, icon: $icon_meter }

```
