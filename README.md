# wifi-locator-model
ML model to locate mobile devices around the house based on wifi signal values.

# Getting the wifi signal data
The easiest way I've found to collect wifi signal strength data was using the [Home Assistant unifi component](https://github.com/home-assistant/home-assistant/tree/dev/homeassistant/components/unifi). Once set up you will get two values related to the signal strength (rssi, signal) as well as the MAC address of the access point the device is connected to. The component updates every 30 seconds, making it fairly easy to collect data.

# Collecting data
The export_to_csv.py script can be used to write collected data to a csv file. The values can be passed to the script as parameters:
`python /config/export_to_csv.py -r {{rssi}} -s {{signal}} -a {{ap}} -l {{location}}`

The location parameter is the label of the current location/room.

# Labeling the data
To label the data using Home Assistant, add a new input_select with the different rooms:
```
input_select:
    phone_location:
    name: Phone location
    options:
        - Living Room
        - Kitchen
        - Bedroom
        - Office
        - Jere Room
        - Bathroom
    initial: Living Room
    icon: mdi:cellphone
```
You'll then be able to manually change those values in the UI during data collection.

# Automating the data collection
Automating the data collection is fairly easy using Home assistant and/or Node-red. Step one is to add the export script as a HA shell command like this:
```
shell_command:
  export_csv: 'python /config/export_to_csv.py -r {{rssi}} -s {{signal}} -a {{ap}} -l {{location}}'
```

If you'd like to include the values directly in this shell command using templates, that can be done as well. For example `export_csv: 'python /config/export_to_csv.py -r {{ states.device_tracker.phone.attributes.rssi }} -s {{ states.device_tracker.phone.attributes.rssi }} -a {{ states.device_tracker.phone.attributes.rssi }} -l {{ states.inpout_select.phone_location.state }}'`

Using the shell_command, data collection can then be automated. Whenever the device_tracker updates with new wifi signal information, call the shell_command to write those to your CSV file. Here's an example of how this is done using Node-red:

```
[{"id":"6d2b985d.4e02d8","type":"api-render-template","z":"8d972e1e.28c54","name":"Signal template","server":"90ba4a50.2b5e48","template":"{% set device = states.device_tracker.sevenplus_router %}\n{\"rssi\":{{device.attributes.rssi}},\"signal\":{{device.attributes.signal}},\"ap\":\"{{device.attributes.ap_mac}}\",\"location\":\"{{ states.input_select.phone_location.state }}\"}","x":340,"y":320,"wires":[["729009fd.14d4d8"]]},{"id":"c7ae5708.04f1b8","type":"trigger-state","z":"8d972e1e.28c54","name":"Changed","server":"90ba4a50.2b5e48","entityid":"device_tracker.sevenplus_router","entityidfiltertype":"exact","debugenabled":false,"constraints":[{"id":"26xjecbhtywk","targetType":"entity_id","targetValue":"input_boolean.training_mode","propertyType":"current_state","propertyValue":"new_state.state","comparatorType":"is","comparatorValueDatatype":"str","comparatorValue":"on"}],"constraintsmustmatch":"all","outputs":2,"customoutputs":[],"outputinitially":false,"state_type":"str","x":120,"y":320,"wires":[["6d2b985d.4e02d8"],[]]},{"id":"729009fd.14d4d8","type":"function","z":"8d972e1e.28c54","name":"Template","func":"var dataString = JSON.stringify(msg.payload);\nmsg.payload = \"{\\\"data\\\":\"+dataString+\"}\";\nreturn msg;","outputs":1,"noerr":0,"x":535,"y":318,"wires":[["5737b248.a211fc"]]},{"id":"5737b248.a211fc","type":"api-call-service","z":"8d972e1e.28c54","name":"","server":"90ba4a50.2b5e48","service_domain":"shell_command","service":"export_csv","data":"","mergecontext":"","x":810,"y":340,"wires":[[]]},{"id":"90ba4a50.2b5e48","type":"server","z":"","name":"HomeAssistant","legacy":false}]
```

Make sure to set the proper device_tracker and input_select names in the template. 

# Adding things to the UI
To make training easier, I added everything to my Home Assistant Lovelace UI:
```
  - id: 9
    title: Training
    icon: mdi:library-books
    cards:
      - id: training  # Automatically created id
        type: entities
        entities:
          - input_boolean.training_mode
          - input_select.phone_location
```

Note the input_boolean.training_mode, it's essentially a switch that will start and stop the data collection. It might be useful to add an additional automation to switch off training_mode after 30 minutes or an hour. It happened to me a couple of times that I turned on the training mode and then just walked away, ruining the collected data. Doing data collection 30 minutes at a time can help with that.