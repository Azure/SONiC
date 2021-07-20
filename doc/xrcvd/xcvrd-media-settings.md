# Proposed Changes to Xcvrd
## Introdcution
### Media settings key
- Before
  - get_media_settings_key() returns a list of 2 keys (strings)
  - get_media_settings_value() expects list of 2 keys
- After
  - get_media_settings_key() returns a list of N strings
    - Return 2 existing keys (part-number and generic), plus new generic key
      - New generic key: form_factor + media_interface [ + cable_length_detailed ]
      - Maintains backward compatibility with media_settings.json for existing platforms
  - get_media_settings_value() accepts list of N strings (keys)
    - Attempts match with each member of key list
### Setting interface type
- Before
  
- After



