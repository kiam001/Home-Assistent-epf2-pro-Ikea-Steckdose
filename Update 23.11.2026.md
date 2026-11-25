Update 23.11.2026

YAML /config/configuration.yaml

```
input_number:
  scooter_referenz_reichweite_25c:
    name: "Scooter Referenz-Reichweite (25°C)"
    min: 10
    max: 120
    unit_of_measurement: "km"
    initial: 58

  epfskate_referenz_reichweite_25c:
    name: "Scooter Referenz-Reichweite (25°C)"
    min: 10
    max: 120
    unit_of_measurement: "km"
    initial: 35
    
sensor:
  - platform: derivative
    source: sensor.grillplats_plug_leistung_3
    name: "Scooter Charger Power Derivative"
    unique_id: scooter_charger_power_derivative
    unit_time: min
    time_window: "00:03:00"


  - platform: derivative
    source: sensor.schlafzimmer_escooter_schnelllader_leistung
    name: "Scooter Schnellader Charger Power Derivative"
    unique_id: scooter_schnelllader_charger_power_derivative
    unit_time: min
    time_window: "00:03:00"


  - platform: derivative
    source: sensor.schlafzimmer_epfsakate_leistung
    name: "EPF Skate Charger Power Derivative"
    unique_id: schlafzimmer_epfsakate_charger_power_derivative
    unit_time: min
    time_window: "00:03:00"


template:
  - sensor:
      # 1. Sensor für die geladenen Kilometer (basierend auf 12,6 Wh/km)
      - name: "Scooter Geladene Kilometer"
        unique_id: scooter_km_session
        unit_of_measurement: "km"
        device_class: distance
        state_class: measurement
        state: >
          {# Berechnung: kWh * 79.36 (entspricht 12.6 Wh/km) #}
          {{ (states('sensor.e_scooter1_scooter_energy_session') | float(0) * 79.36) | round(2) }}
        attributes:
          wh_per_km: 12.6
          calculation_factor: 79.36

      # 2. Sensor für die Ladekosten (mit festen 2 Nachkommastellen)
      - name: "Scooter Ladekosten"
        unique_id: scooter_cost_session
        unit_of_measurement: "€"
        device_class: monetary
        state: >
          {# Preis: 0.39 € pro kWh | Formatierung auf 2 Stellen #}
          {% set cost = states('sensor.e_scooter1_scooter_energy_session') | float(0) * 0.39 %}
          {{ "{:.2f}".format(cost) }}

      - name: "Scooter Referenz-Reichweite aktuell"
        unique_id: scooter_referenz_reichweite_aktuell
        unit_of_measurement: "km"
        device_class: distance
        state: >
          {% set t = state_attr('weather.forecast_home', 'temperature') | float(25) %}
          {% set range_25c = states('input_number.scooter_referenz_reichweite_25c') | float(50) %}
          {{ (range_25c * (0.01538 * t + 0.61538)) | round(1) }}

  - sensor:
      - name: "Scooter Aktuelle Reichweite (geschätzt)"
        unique_id: scooter_range_estimated
        unit_of_measurement: "km"
        device_class: distance
        state: >
          {% set soc = states('sensor.scooter_akku_prozent') | float(0) %}
          {% set ref_range = states('sensor.scooter_referenz_reichweite_aktuell') | float(50) %}
          {{ ((soc / 100) * ref_range) | round(0) }}

  - binary_sensor:
      - name: "Scooter Ladevorgang Stabil"
        unique_id: scooter_charging_stable
        delay_on: "00:00:10"
        state: "{{ states('sensor.grillplats_plug_leistung_3') | float(0) > 10 }}"
    sensor:
      - name: "Scooter Akku Spannung (geschätzt)"
        unique_id: scooter_voltage_estimated
        unit_of_measurement: "V"
        device_class: voltage
        state: >
          {% set watt = states('sensor.grillplats_plug_leistung_3') | float(0) %}
          {% set last_v = this.state | float(0) %}
          {% set is_stable = is_state('binary_sensor.scooter_ladevorgang_stabil', 'on') %}
          {% if watt > 10 %}
            {% if is_stable %}
              {% if watt > 130 %}
                {# CC-Phase: Wir berechnen den Wert und halten den Höchststand #}
                {% set current_calc = (watt * 0.2957) | round(1) %}
                {% set peak = [current_calc, last_v] | max %}
                {# Das hier ist die sichere Schreibweise für das Limit von 54.4V #}
                {{ [peak, 54.4] | min }}
              {% else %}
                {# CV-Phase: Watt sinken am Ende #}
                54.4
              {% endif %}
            {% else %}
              {# Während der 10s Einschaltphase halten wir 0 oder den alten Wert #}
              {{ last_v if last_v > 0 else 0 }}
            {% endif %}
          {% else %}
            {# Reset wenn Stecker gezogen #}
            0
          {% endif %}

  # 1. DER ANKER (Trigger-basiert)
  # Er merkt sich EINMALIG beim Einstecken, wie voll der Akku war.
  - trigger:
      - platform: numeric_state
        entity_id: sensor.grillplats_plug_leistung_3
        above: 10
        for: "00:01:00"
    sensor:
      - name: "Scooter Start Reichweite Fix"
        unique_id: scooter_start_range_fix
        unit_of_measurement: "km"
        state: >
          {% set soc = states('sensor.scooter_akku_prozent') | float(0) %}
          {% set ref_range = states('sensor.scooter_referenz_reichweite_aktuell') | float(50) %}
          {{ ((soc / 100) * ref_range) | round(0) }}

  # 2. DIE LIVE-ANZEIGE (Normaler Template-Sensor)
  # Dieser Sensor rechnet bei jeder Änderung der geladenen km sofort neu.
  - sensor:
      - name: "Scooter Dynamische Gesamt-Reichweite"
        unique_id: scooter_total_range_dynamic
        unit_of_measurement: "km"
        device_class: distance
        state: >
          {% set start_km = states('sensor.scooter_start_reichweite_fix') | float(0) %}
          {% set geladen_km = states('sensor.scooter_geladene_kilometer') | float(0) %}
          {% set max_km = states('sensor.scooter_referenz_reichweite_aktuell') | float(50) %}
          {% set gesamt = (start_km + geladen_km) | round(1) %}
          {{ [gesamt, max_km] | min }}

  - sensor:
      - name: "Scooter Akku Prozent"
        unique_id: scooter_battery_level
        unit_of_measurement: "%"
        device_class: battery
        state_class: measurement
        state: >
          {%- set charger_current = 3.0 -%}
          {%- set charger_efficiency = 92 -%}
          {%- set cc_min_power = 130.0 -%}
          {%- set derive = states('sensor.scooter_charger_power_derivative') | float(none) -%}
          {%- set charger_power = states('sensor.grillplats_plug_leistung_3') | float(none) -%}
          {%- set is_stable = is_state('binary_sensor.scooter_ladevorgang_stabil', 'on') -%}
          {%- if charger_power != none and charger_power > 10 and is_stable and derive != none -%}
          {%-   if charger_power >= cc_min_power and derive > -0.7 -%}
          {%-     set v_est = (charger_power * (charger_efficiency / 100)) / charger_current -%}
          {%-     set table = [
                    [39.70, 0], [41.40, 5], [43.20, 10], [45.20, 20], [46.65, 30],
                    [47.95, 40], [49.15, 50], [50.45, 60], [51.75, 70], [53.15, 80],
                    [54.60, 90]
                  ] -%}
          {%-     set ns = namespace(soc=0, found=false) -%}
          {%-     if v_est <= table[0][0] -%}
          {%-       set ns.soc = 0 -%}
          {%-     elif v_est >= table[-1][0] -%}
          {%-       set ns.soc = 90 -%}
          {%-     else -%}
          {%-       for i in range(table|length - 1) -%}
          {%-         if not ns.found and v_est >= table[i][0] and v_est <= table[i+1][0] -%}
          {%-           set v1, s1 = table[i][0], table[i][1] -%}
          {%-           set v2, s2 = table[i+1][0], table[i+1][1] -%}
          {%-           set ns.soc = s1 + (v_est - v1) * (s2 - s1) / (v2 - v1) -%}
          {%-           set ns.found = true -%}
          {%-         endif -%}
          {%-       endfor -%}
          {%-     endif -%}
          {{      ns.soc | round(0) }}
          {%-   else -%}
          {%-     set p_start = (54.6 * charger_current) / (charger_efficiency / 100) -%}
          {%-     set p_end = 15.0 -%}
          {%-     if charger_power >= p_start -%}
          {{        90 }}
          {%-     elif charger_power <= p_end -%}
          {{        100 }}
          {%-     else -%}
          {{        (100.0 - ((charger_power - p_end) / (p_start - p_end)) * 10.0) | round(0) }}
          {%-     endif -%}
          {%-   endif -%}
          {%- else -%}
          {{    (this.state | float(0)) | round(0) if this is defined else 0 }}
          {%- endif -%}

          
  - sensor:
      # 1. Sensor für die geladenen Kilometer (basierend auf 12,6 Wh/km)
      - name: "Scooter Schnelllader Geladene Kilometer"
        unique_id: scooter_schnelllader_km_session
        unit_of_measurement: "km"
        device_class: distance
        state_class: measurement
        state: >
          {# Berechnung: kWh * 79.36 (entspricht 12.6 Wh/km) #}
          {{ (states('sensor.schlafzimmer_escooter_schnelllader_schnellader_geladene_energie') | float(0) * 79.36) | round(2) }}
        attributes:
          wh_per_km: 12.6
          calculation_factor: 79.36

      # 2. Sensor für die Ladekosten (mit festen 2 Nachkommastellen)
      - name: "Scooter Schnelllader Ladekosten"
        unique_id: scooter_schnelllader_cost_session
        unit_of_measurement: "€"
        device_class: monetary
        state: >
          {# Preis: 0.39 € pro kWh | Formatierung auf 2 Stellen #}
          {% set cost = states('sensor.schlafzimmer_escooter_schnelllader_schnellader_geladene_energie') | float(0) * 0.39 %}
          {{ "{:.2f}".format(cost) }}

      - name: "Scooter Schnelllader Referenz-Reichweite aktuell"
        unique_id: scooter_schnelllader_referenz_reichweite_aktuell
        unit_of_measurement: "km"
        device_class: distance
        state: >
          {% set t = state_attr('weather.forecast_home','temperature') | float(25) %}
          {% set range_25c = states('input_number.scooter_referenz_reichweite_25c') | float(50) %}
          {{ (range_25c * (0.01538 * t + 0.61538)) | round(1) }}

  - sensor:
      - name: "Scooter Schnelllader Aktuelle Reichweite (geschätzt)"
        unique_id: scooter_schnelllader_range_estimated
        unit_of_measurement: "km"
        device_class: distance
        state: >
          {% set soc = states('sensor.scooter_schnelllader_akku_prozent') | float(0) %}
          {% set ref_range = states('sensor.scooter_schnelllader_referenz_reichweite_aktuell') | float(50) %}
          {{ ((soc / 100) * ref_range) | round(0) }}

  - binary_sensor:
      - name: "Scooter Schnelllader Ladevorgang Stabil"
        unique_id: scooter_schnelllader_charging_stable
        delay_on: "00:00:10"
        state: "{{ states('sensor.schlafzimmer_escooter_schnelllader_leistung') | float(0) > 10 }}"
    sensor:
      - name: "Scooter Schnelllader Akku Spannung (geschätzt)"
        unique_id: scooter_schnelllader_voltage_estimated
        unit_of_measurement: "V"
        device_class: voltage
        state: >
          {% set watt = states('sensor.schlafzimmer_escooter_schnelllader_leistung') | float(0) %}
          {% set last_v = this.state | float(0) %}
          {% set is_stable = is_state('binary_sensor.scooter_schnelllader_ladevorgang_stabil', 'on') %}
          {% if watt > 10 %}
            {% if is_stable %}
              {% if watt > 195 %}
                {# CC-Phase (4,5A): Schwelle auf 195W und Faktor auf 0.1971 angepasst #}
                {% set current_calc = (watt * 0.1971) | round(1) %}
                {% set peak = [current_calc, last_v] | max %}
                {# Limit von 54.4V #}
                {{ [peak, 54.4] | min }}
              {% else %}
                {# CV-Phase: Watt sinken am Ende unter 195W #}
                54.4
              {% endif %}
            {% else %}
              {# Während der 10s Einschaltphase halten wir 0 oder den alten Wert #}
              {{ last_v if last_v > 0 else 0 }}
            {% endif %}
          {% else %}
            {# Reset wenn Stecker gezogen #}
            0
          {% endif %}

  # 1. DER ANKER (Trigger-basiert)
  # Er merkt sich EINMALIG beim Einstecken, wie voll der Akku war.
  - trigger:
      - platform: numeric_state
        entity_id: sensor.schlafzimmer_escooter_schnelllader_leistung
        above: 10
        for: "00:01:00"
    sensor:
      - name: "Scooter Schnelllader Start Reichweite Fix"
        unique_id: scooter_schnelllader_start_range_fix
        unit_of_measurement: "km"
        state: >
          {% set soc = states('sensor.scooter_schnelllader_akku_prozent') | float(0) %}
          {% set ref_range = states('sensor.scooter_schnelllader_referenz_reichweite_aktuell') | float(50) %}
          {{ ((soc / 100) * ref_range) | round(0) }}

  # 2. DIE LIVE-ANZEIGE (Normaler Template-Sensor)
  # Dieser Sensor rechnet bei jeder Änderung der geladenen km sofort neu.
  - sensor:
      - name: "Scooter Schnelllader Dynamische Gesamt-Reichweite"
        unique_id: scooter_schnelllader_total_range_dynamic
        unit_of_measurement: "km"
        device_class: distance
        state: >
          {% set start_km = states('sensor.scooter_schnelllader_start_reichweite_fix') | float(0) %}
          {% set geladen_km = states('sensor.scooter_schnelllader_geladene_kilometer') | float(0) %}
          {% set max_km = states('sensor.scooter_schnelllader_referenz_reichweite_aktuell') | float(50) %}
          {% set gesamt = (start_km + geladen_km) | round(1) %}
          {{ [gesamt, max_km] | min }}

  - sensor:
      - name: "Scooter Schnelllader Akku Prozent"
        unique_id: scooter_schnelllader_battery_level
        unit_of_measurement: "%"
        device_class: battery
        state_class: measurement
        state: >
          {%- set charger_current = 4.5 -%}
          {%- set charger_efficiency = 92 -%}
          {%- set derive = states('sensor.scooter_schnelllader_charger_power_derivative') | float(none) -%}
          {%- set charger_power = states('sensor.schlafzimmer_escooter_schnelllader_leistung') | float(none) -%}
          {%- if charger_power != none and charger_power > 5 and derive != none -%}
          {%-   if derive > 0 -%}
          {%-     set v_est = (charger_power * (charger_efficiency / 100)) / charger_current -%}
          {%-     set table = [
                    [39.70, 0], [41.40, 5], [43.20, 10], [45.20, 20], [46.65, 30],
                    [47.95, 40], [49.15, 50], [50.45, 60], [51.75, 70], [53.15, 80],
                    [54.60, 90]
                  ] -%}
          {%-     set ns = namespace(soc=0, found=false) -%}
          {%-     if v_est <= table[0][0] -%}
          {%-       set ns.soc = 0 -%}
          {%-     elif v_est >= table[-1][0] -%}
          {%-       set ns.soc = 90 -%}
          {%-     else -%}
          {%-       for i in range(table|length - 1) -%}
          {%-         if not ns.found and v_est >= table[i][0] and v_est <= table[i+1][0] -%}
          {%-           set v1, s1 = table[i][0], table[i][1] -%}
          {%-           set v2, s2 = table[i+1][0], table[i+1][1] -%}
          {%-           set ns.soc = s1 + (v_est - v1) * (s2 - s1) / (v2 - v1) -%}
          {%-           set ns.found = true -%}
          {%-         endif -%}
          {%-       endfor -%}
          {%-     endif -%}
          {{      ns.soc | round(0) }}
          {%-   else -%}
          {%-     set p_start = (54.6 * charger_current) / (charger_efficiency / 100) -%}
          {%-     set p_end = 15.0 -%}
          {%-     if charger_power >= p_start -%}
          {{        90 }}
          {%-     elif charger_power <= p_end -%}
          {{        100 }}
          {%-     else -%}
          {{        (100.0 - ((charger_power - p_end) / (p_start - p_end)) * 10.0) | round(0) }}
          {%-     endif -%}
          {%-   endif -%}
          {%- else -%}
          {{    (this.state | float(0)) | round(0) if this is defined else 0 }}
          {%- endif -%}

  - sensor:
      # 1. Sensor für die geladenen Kilometer (basierend auf 14,39 Wh/km ab Steckdose)
      - name: "EPF Skate Geladene Kilometer"
        unique_id: schlafzimmer_epfsakate_km_session
        unit_of_measurement: "km"
        device_class: distance
        state_class: measurement
        state: >
          {# Berechnung: kWh * 69.5 (entspricht 14.39 Wh/km inkl. ~8% Ladeverlusten bei 432 Wh / 32.6 km) #}
          {{ (states('sensor.schlafzimmer_epfsakate_epfskateenergie') | float(0) * 69.5) | round(2) }}
        attributes:
          wh_per_km: 14.39
          calculation_factor: 69.5

      # 2. Sensor für die Ladekosten (mit festen 2 Nachkommastellen)
      - name: "EPF Skate Ladekosten"
        unique_id: schlafzimmer_epfsakate_cost_session
        unit_of_measurement: "€"
        device_class: monetary
        state: >
          {# Preis: 0.39 € pro kWh | Formatierung auf 2 Stellen #}
          {% set cost = states('sensor.schlafzimmer_epfsakate_epfskateenergie') | float(0) * 0.39 %}
          {{ "{:.2f}".format(cost) }}

      - name: "EPF Skate Referenz-Reichweite aktuell"
        unique_id: schlafzimmer_epfsakate_referenz_reichweite_aktuell
        unit_of_measurement: "km"
        device_class: distance
        state: >
          {% set t = state_attr('weather.forecast_home', 'temperature') | float(25) %}
          {% set range_25c = states('input_number.epfskate_referenz_reichweite_25c') | float(32.6) %}
          {{ (range_25c * (0.01538 * t + 0.61538)) | round(1) }}

  - sensor:
      - name: "EPF Skate Aktuelle Reichweite (geschätzt)"
        unique_id: schlafzimmer_epfsakate_range_estimated
        unit_of_measurement: "km"
        device_class: distance
        state: >
          {% set soc = states('sensor.epf_skate_akku_prozent') | float(0) %}
          {% set ref_range = states('sensor.epf_skate_referenz_reichweite_aktuell') | float(32.6) %}
          {{ ((soc / 100) * ref_range) | round(0) }}

  - binary_sensor:
      - name: "EPF Skate Ladevorgang Stabil"
        unique_id: schlafzimmer_epfsakate_charging_stable
        delay_on: "00:00:10"
        state: "{{ states('sensor.schlafzimmer_epfsakate_leistung') | float(0) > 5 }}"
    sensor:
      - name: "EPF Skate Akku Spannung (geschätzt)"
        unique_id: schlafzimmer_epfsakate_voltage_estimated
        unit_of_measurement: "V"
        device_class: voltage
        state: >
          {% set watt = states('sensor.schlafzimmer_epfsakate_leistung') | float(0) %}
          {% set last_v = this.state | float(0) %}
          {% set is_stable = is_state('binary_sensor.epf_skate_ladevorgang_stabil', 'on') %}
          {% if watt > 5 %}
            {% if is_stable %}
              {% if watt > 85 %}
                {# CC-Phase (2A, ~92% Wirkungsgrad): 0.92 / 2.0 = 0.46 #}
                {% set current_calc = (watt * 0.46) | round(1) %}
                {% set peak = [current_calc, last_v] | max %}
                {# Limit von 54.6V laut Akku-Typenschild #}
                {{ [peak, 54.6] | min }}
              {% else %}
                {# CV-Phase: Leistung sinkt am Ende unter 85W #}
                54.6
              {% endif %}
            {% else %}
              {# Während der 10s Einschaltphase alten Wert halten #}
              {{ last_v if last_v > 0 else 0 }}
            {% endif %}
          {% else %}
            {# Reset wenn Stecker gezogen oder Standby #}
            0
          {% endif %}

  # 1. DER ANKER (Trigger-basiert)
  # Merkt sich EINMALIG beim Einstecken, wie voll der Akku war.
  - trigger:
      - platform: numeric_state
        entity_id: sensor.schlafzimmer_epfsakate_leistung
        above: 5
        for: "00:01:00"
    sensor:
      - name: "EPF Skate Start Reichweite Fix"
        unique_id: schlafzimmer_epfsakate_start_range_fix
        unit_of_measurement: "km"
        state: >
          {% set soc = states('sensor.epf_skate_akku_prozent') | float(0) %}
          {% set ref_range = states('sensor.epf_skate_referenz_reichweite_aktuell') | float(32.6) %}
          {{ ((soc / 100) * ref_range) | round(0) }}

  # 2. DIE LIVE-ANZEIGE (Normaler Template-Sensor)
  # Rechnet bei jeder Änderung der geladenen km sofort neu.
  - sensor:
      - name: "EPF Skate Dynamische Gesamt-Reichweite"
        unique_id: schlafzimmer_epfsakate_total_range_dynamic
        unit_of_measurement: "km"
        device_class: distance
        state: >
          {% set start_km = states('sensor.epf_skate_start_reichweite_fix') | float(0) %}
          {% set geladen_km = states('sensor.epf_skate_geladene_kilometer') | float(0) %}
          {% set max_km = states('sensor.epf_skate_referenz_reichweite_aktuell') | float(32.6) %}
          {% set gesamt = (start_km + geladen_km) | round(1) %}
          {{ [gesamt, max_km] | min }}

  - sensor:
      - name: "EPF Skate Akku Prozent"
        unique_id: schlafzimmer_epfsakate_battery_level
        unit_of_measurement: "%"
        device_class: battery
        state_class: measurement
        state: >
          {%- set charger_current = 2.0 -%}
          {%- set charger_efficiency = 92 -%}
          {%- set cc_min_power = 85.0 -%}
          {%- set derive = states('sensor.epf_skate_charger_power_derivative') | float(none) -%}
          {%- set charger_power = states('sensor.schlafzimmer_epfsakate_leistung') | float(none) -%}
          {%- set is_stable = is_state('binary_sensor.epf_skate_ladevorgang_stabil', 'on') -%}
          {%- if charger_power != none and charger_power > 5 and is_stable and derive != none -%}
          {%-   if charger_power >= cc_min_power and derive > -0.5 -%}
          {%-     set v_est = (charger_power * (charger_efficiency / 100)) / charger_current -%}
          {%-     set table = [
                    [39.70, 0], [41.40, 5], [43.20, 10], [45.20, 20], [46.65, 30],
                    [47.95, 40], [49.15, 50], [50.45, 60], [51.75, 70], [53.15, 80],
                    [54.60, 90]
                  ] -%}
          {%-     set ns = namespace(soc=0, found=false) -%}
          {%-     if v_est <= table[0][0] -%}
          {%-       set ns.soc = 0 -%}
          {%-     elif v_est >= table[-1][0] -%}
          {%-       set ns.soc = 90 -%}
          {%-     else -%}
          {%-       for i in range(table|length - 1) -%}
          {%-         if not ns.found and v_est >= table[i][0] and v_est <= table[i+1][0] -%}
          {%-           set v1, s1 = table[i][0], table[i][1] -%}
          {%-           set v2, s2 = table[i+1][0], table[i+1][1] -%}
          {%-           set ns.soc = s1 + (v_est - v1) * (s2 - s1) / (v2 - v1) -%}
          {%-           set ns.found = true -%}
          {%-         endif -%}
          {%-       endfor -%}
          {%-     endif -%}
          {{      ns.soc | round(0) }}
          {%-   else -%}
          {%-     set p_start = (54.6 * charger_current) / (charger_efficiency / 100) -%}
          {%-     set p_end = 15.0 -%}
          {%-     if charger_power >= p_start -%}
          {{        90 }}
          {%-     elif charger_power <= p_end -%}
          {{        100 }}
          {%-     else -%}
          {{        (100.0 - ((charger_power - p_end) / (p_start - p_end)) * 10.0) | round(0) }}
          {%-     endif -%}
          {%-   endif -%}
          {%- else -%}
          {{    (this.state | float(0)) | round(0) if this is defined else 0 }}
          {%- endif -%}
```

Dashboard:

```
views:
  - title: E-Scooter Ladestation
    path: e-scooter-ladestation
    icon: mdi:scooter-electric
    type: sections
    max_columns: 3
    badges:
      - type: entity
        entity: input_boolean.akkulagern
        name: Akku-Lagermodus
        icon: mdi:battery-heart-variant
        show_name: true
        show_state: true
        color: orange
        tap_action:
          action: toggle
      - type: entity
        entity: switch.schlafzimmer_escooter_schnelllader
        name: ePF-2 PRO (4,5 A)
        icon: mdi:lightning-bolt-circle
        show_name: true
        show_state: true
        color: amber
        tap_action:
          action: toggle
      - type: entity
        entity: switch.grillplats_plug_3
        name: ePF-2 PRO (3 A)
        icon: mdi:power-plug
        show_name: true
        show_state: true
        color: blue
        tap_action:
          action: toggle
      - type: entity
        entity: switch.schlafzimmer_epfsakate
        name: EPF Skate (2 A)
        icon: mdi:skateboard
        show_name: true
        show_state: true
        color: green
        tap_action:
          action: toggle
    sections:
      - type: grid
        background: true
        cards:
          - type: heading
            heading: ePF-2 PRO · 4,5 A
            heading_style: title
            icon: mdi:lightning-bolt-circle
            badges:
              - type: entity
                entity: sensor.schlafzimmer_escooter_schnelllader_leistung
                show_state: true
                color: amber
              - type: entity
                entity: sensor.scooter_schnelllader_akku_spannung_geschatzt
                show_state: true
                color: blue
          - type: gauge
            entity: sensor.scooter_schnelllader_akku_prozent
            name: Akkustand
            min: 0
            max: 100
            needle: true
            segments:
              - from: 0
                color: '#db4437'
              - from: 20
                color: '#ffa600'
              - from: 50
                color: '#039be5'
              - from: 80
                color: '#43a047'
            grid_options:
              columns: 6
              rows: 3
          - type: sensor
            entity: sensor.schlafzimmer_escooter_schnelllader_leistung
            name: Ladeleistung
            icon: mdi:flash
            graph: line
            hours_to_show: 4
            detail: 2
            grid_options:
              columns: 6
              rows: 3
          - type: tile
            entity: switch.schlafzimmer_escooter_schnelllader
            name: Ladegerät
            icon: mdi:power-plug
            color: amber
          - type: tile
            entity: input_button.scooter2res
            name: Session Reset
            icon: mdi:restart
            color: red
          - type: heading
            heading: Reichweite & Akku
            heading_style: subtitle
            icon: mdi:map-marker-distance
          - type: tile
            entity: sensor.scooter_schnelllader_dynamische_gesamt_reichweite
            name: Reichweite (Live)
            icon: mdi:map-marker-distance
            color: green
          - type: tile
            entity: sensor.scooter_schnelllader_aktuelle_reichweite_geschatzt
            name: Reichweite (SoC)
            icon: mdi:navigation-variant
            color: teal
          - type: tile
            entity: sensor.scooter_schnelllader_akku_spannung_geschatzt
            name: Spannung
            icon: mdi:sine-wave
            color: blue
          - type: tile
            entity: sensor.scooter_schnelllader_referenz_reichweite_aktuell
            name: Max. heute
            icon: mdi:thermometer
            color: cyan
          - type: heading
            heading: Ladesitzung
            heading_style: subtitle
            icon: mdi:chart-areaspline
          - type: tile
            entity: sensor.scooter_schnelllader_geladene_kilometer
            name: Nachgeladen
            icon: mdi:plus-circle-outline
            color: light-green
          - type: tile
            entity: sensor.schlafzimmer_escooter_schnelllader_schnellader_geladene_energie
            name: Energie
            icon: mdi:meter-electric
            color: indigo
          - type: tile
            entity: sensor.scooter_schnelllader_ladekosten
            name: Ladekosten
            icon: mdi:currency-eur
            color: purple
          - type: tile
            entity: sensor.scooter_schnelllader_charger_power_derivative
            name: Trend (W/min)
            icon: mdi:chart-line-variant
            color: grey
      - type: grid
        background: true
        cards:
          - type: heading
            heading: ePF-2 PRO · 3 A
            heading_style: title
            icon: mdi:scooter-electric
            badges:
              - type: entity
                entity: sensor.grillplats_plug_leistung_3
                show_state: true
                color: amber
              - type: entity
                entity: sensor.scooter_akku_spannung_geschatzt
                show_state: true
                color: blue
          - type: gauge
            entity: sensor.scooter_akku_prozent
            name: Akkustand
            min: 0
            max: 100
            needle: true
            segments:
              - from: 0
                color: '#db4437'
              - from: 20
                color: '#ffa600'
              - from: 50
                color: '#039be5'
              - from: 80
                color: '#43a047'
            grid_options:
              columns: 6
              rows: 3
          - type: sensor
            entity: sensor.grillplats_plug_leistung_3
            name: Ladeleistung
            icon: mdi:flash
            graph: line
            hours_to_show: 4
            detail: 2
            grid_options:
              columns: 6
              rows: 3
          - type: tile
            entity: switch.grillplats_plug_3
            name: Ladegerät
            icon: mdi:power-plug
            color: blue
          - type: tile
            entity: input_button.scooter1reser
            name: Session Reset
            icon: mdi:restart
            color: red
          - type: heading
            heading: Reichweite & Akku
            heading_style: subtitle
            icon: mdi:map-marker-distance
          - type: tile
            entity: sensor.scooter_dynamische_gesamt_reichweite
            name: Reichweite (Live)
            icon: mdi:map-marker-distance
            color: green
          - type: tile
            entity: sensor.scooter_aktuelle_reichweite_geschatzt
            name: Reichweite (SoC)
            icon: mdi:navigation-variant
            color: teal
          - type: tile
            entity: sensor.scooter_akku_spannung_geschatzt
            name: Spannung
            icon: mdi:sine-wave
            color: blue
          - type: tile
            entity: sensor.scooter_referenz_reichweite_aktuell
            name: Max. heute
            icon: mdi:thermometer
            color: cyan
          - type: heading
            heading: Ladesitzung
            heading_style: subtitle
            icon: mdi:chart-areaspline
          - type: tile
            entity: sensor.scooter_geladene_kilometer
            name: Nachgeladen
            icon: mdi:plus-circle-outline
            color: light-green
          - type: tile
            entity: sensor.e_scooter1_scooter_energy_session
            name: Energie
            icon: mdi:meter-electric
            color: indigo
          - type: tile
            entity: sensor.scooter_ladekosten
            name: Ladekosten
            icon: mdi:currency-eur
            color: purple
          - type: tile
            entity: sensor.scooter_charger_power_derivative
            name: Trend (W/min)
            icon: mdi:chart-line-variant
            color: grey
      - type: grid
        background: true
        cards:
          - type: heading
            heading: EPF Skate · 2 A
            heading_style: title
            icon: mdi:skateboard
            badges:
              - type: entity
                entity: sensor.schlafzimmer_epfsakate_leistung
                show_state: true
                color: amber
              - type: entity
                entity: sensor.epf_skate_akku_spannung_geschatzt
                show_state: true
                color: blue
          - type: gauge
            entity: sensor.epf_skate_akku_prozent
            name: Akkustand
            min: 0
            max: 100
            needle: true
            segments:
              - from: 0
                color: '#db4437'
              - from: 20
                color: '#ffa600'
              - from: 50
                color: '#039be5'
              - from: 80
                color: '#43a047'
            grid_options:
              columns: 6
              rows: 3
          - type: sensor
            entity: sensor.schlafzimmer_epfsakate_leistung
            name: Ladeleistung
            icon: mdi:flash
            graph: line
            hours_to_show: 4
            detail: 2
            grid_options:
              columns: 6
              rows: 3
          - type: tile
            entity: switch.schlafzimmer_epfsakate
            name: Ladegerät
            icon: mdi:power-plug
            color: green
          - type: tile
            entity: input_button.epfskatereset
            name: Session Reset
            icon: mdi:restart
            color: red
          - type: heading
            heading: Reichweite & Akku
            heading_style: subtitle
            icon: mdi:map-marker-distance
          - type: tile
            entity: sensor.epf_skate_dynamische_gesamt_reichweite
            name: Reichweite (Live)
            icon: mdi:map-marker-distance
            color: green
          - type: tile
            entity: sensor.epf_skate_aktuelle_reichweite_geschatzt
            name: Reichweite (SoC)
            icon: mdi:navigation-variant
            color: teal
          - type: tile
            entity: sensor.epf_skate_akku_spannung_geschatzt
            name: Spannung
            icon: mdi:sine-wave
            color: blue
          - type: tile
            entity: sensor.epf_skate_referenz_reichweite_aktuell
            name: Max. heute
            icon: mdi:thermometer
            color: cyan
          - type: heading
            heading: Ladesitzung
            heading_style: subtitle
            icon: mdi:chart-areaspline
          - type: tile
            entity: sensor.epf_skate_geladene_kilometer
            name: Nachgeladen
            icon: mdi:plus-circle-outline
            color: light-green
          - type: tile
            entity: sensor.schlafzimmer_epfsakate_epfskateenergie
            name: Energie
            icon: mdi:meter-electric
            color: indigo
          - type: tile
            entity: sensor.epf_skate_ladekosten
            name: Ladekosten
            icon: mdi:currency-eur
            color: purple
          - type: tile
            entity: sensor.epf_skate_charger_power_derivative
            name: Trend (W/min)
            icon: mdi:chart-line-variant
            color: grey
      - type: grid
        column_span: 3
        background: true
        cards:
          - type: heading
            heading: Einstellungen & Referenzwerte (25 °C)
            heading_style: title
            icon: mdi:tune
            grid_options:
              columns: full
          - type: tile
            entity: input_boolean.akkulagern
            name: Akku-Lagermodus
            icon: mdi:battery-heart-variant
            color: orange
            grid_options:
              columns: 12
          - type: tile
            entity: input_number.scooter_referenz_reichweite_25c
            name: ePF-2 PRO Referenz (25 °C)
            icon: mdi:scooter-electric
            color: blue
            features_position: bottom
            features:
              - type: numeric-input
                style: slider
            grid_options:
              columns: 12
          - type: tile
            entity: input_number.epfskate_referenz_reichweite_25c
            name: EPF Skate Referenz (25 °C)
            icon: mdi:skateboard
            color: green
            features_position: bottom
            features:
              - type: numeric-input
                style: slider
            grid_options:
              columns: 12
```
