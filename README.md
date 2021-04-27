# Mailbox-Monitor
Home Assistant automation for state monitoring of a physical mailbox

I have a door contact sensor (reed switch) on my mailbox like many of us who've wanted notifications of when physical mail is received. What I found to me missing though was the *state* of the mailbox, specifically if the mail had already been checked and picked up by a member of the house. Additionally, I wanted to avoid ambiguity with mailbox door events for outgoing mail or new mail that had been left overnight and picked up in the morning.

This automation makes a few generalized assumptions:

* Mail is only delivered once per day and never before a certain time of day (11:00am for instance - configurable).  This will get out of sync if you have multiple mail carriers, neighbors dropping off mail, newspaper delivery in your mailbox etc... none of which are issues for me personally

* Similar to above, mail is only checked once a day and presumably always after mail has been delivered.  We assume that those checking the mail will know the state of the mailbox before looking (the whole point of tracking state info).  This may get out of sync if a house-sitter or guest is checking mail during the window when mail might be delivered.

* Outgoing mail is only ever left in the mailbox in the morning, before the typical delivery time and only when a member of the household is home. (If outgoing mail is left during a pickup in the evening, or in the morning while picking up old mail we ignore this edge case and assume the box to still be empty – the status of if the mailbox needs to be checked is the primary motivator for this automation) + as in the last case, outgoing mail is typically very low frequency and can generally be ignored.

* In the case where the mailbox door was left open (> 1min or overnight etc) we assume that the door is only ever closed by someone picking up the (incoming) mail, not from the mail courier, and we can assume the box was just emptied. Thought being that a household member would notice an open door in the morning while leaving, or in the evening during pickup if the door had fallen open during mail delivery. That and the frequency of outgoing mail is low so it’s unlikely that the door would fall open after dropping off outgoing mail.

[u]Logic walkthrough:[/u]

* Automation triggers on mailbox door state changes, or at midnight (to capture new vs old mail left in the mailbox overnight)

* Choose condition for ‘midnight roll over’, activity prior to earliest delivery time, activity after earliest delivery time

Midnight rollover:

1. If there’s still mail in the box (either new or old), it’s now marked as **Incoming:Old**

2. If the door is Left Open, and it’s still open – mark state as **Left Open**

3. Otherwise mark mailbox as **Empty** (to help reset states if things get out of whack)

Morning, prior to earliest delivery and the door is open:

1. If there was mail in the box, someone just picked it up (household member, house sitter etc) – mark state as **Empty**

2. If the box was empty, and an adult is home, assume outgoing mail was just dropped off – mark state as **Outgoing**

3. Default, (ie, an empty mailbox was just checked and is still empty) – mark state as **Empty**

After earliest delivery and the door is open:

1. If there was mail in the mailbox either new or old, it has now just been picked up by someone – mark state as **Empty**

2. In any other case (empty, outgoing, left open [this shouldn’t happen as left open shouldn’t be the current state during a door open trigger event]), mail has just been delivered – mark state as **Incoming:New**

-Default- & checking for left open:

If the door state is open, wait a period of time (1 minute)… if that timer expires, the door has been left open - mark state as **Left Open**. Automation is configured in restart mode, therefore a close door trigger will terminate this wait early, run through the default condition earlier and fail (exit cleanly) on this test without any mailbox state changes

**[u]That's great and all... Just show me the code![/u]**

I wanted to make this a blueprint on the community forums – but blueprints don’t allow for automatic generation of helper variables (and I had issues using !input with input_select variables) so there’s some manual user intervention required. 

I did blueprint what I could though, found here: XXXXXXXXXXXXX
or in its original form below:

[details="Mailbox Monitoring Automation"]
```
alias: Mailbox Monitoring
description: 'mailbox state monitoring'
trigger:
  - platform: state
    entity_id: binary_sensor.mailbox_is_open
  - platform: time
    at: '00:00:00'
condition: []
action:
  - choose:
      - conditions:
          - condition: time
            before: '00:01:00'
        sequence:
          - choose:
              - conditions:
                  - condition: or
                    conditions:
                      - condition: state
                        entity_id: input_select.mailbox_status
                        state: 'Incoming:New'
                      - condition: state
                        entity_id: input_select.mailbox_status
                        state: 'Incoming:Old'
                sequence:
                  - service: input_select.select_option
                    target:
                      entity_id: input_select.mailbox_status
                    data:
                      option: 'Incoming:Old'
              - conditions:
                  - condition: state
                    entity_id: input_select.mailbox_status
                    state: Left Open
                sequence:
                  - service: input_select.select_option
                    target:
                      entity_id: input_select.mailbox_status
                    data:
                      option: Left Open
            default:
              - service: input_select.select_option
                target:
                  entity_id: input_select.mailbox_status
                data:
                  option: Empty
      - conditions:
          - condition: and
            conditions:
              - condition: time
                before: '11:00:00'
              - condition: state
                entity_id: binary_sensor.mailbox_is_open
                state: 'on'
        sequence:
          - choose:
              - conditions:
                  - condition: or
                    conditions:
                      - condition: state
                        entity_id: input_select.mailbox_status
                        state: 'Incoming:New'
                      - condition: state
                        entity_id: input_select.mailbox_status
                        state: 'Incoming:Old'
                sequence:
                  - service: input_select.select_option
                    target:
                      entity_id: input_select.mailbox_status
                    data:
                      option: Empty
              - conditions:
                  - condition: and
                    conditions:
                      - condition: or
                        conditions:
                          - condition: state
                            entity_id: input_select.mailbox_status
                            state: Empty
                          - condition: state
                            entity_id: input_select.mailbox_status
                            state: Outgoing
                      - condition: state
                        entity_id: group.family_adults
                        state: home
                sequence:
                  - service: input_select.select_option
                    target:
                      entity_id: input_select.mailbox_status
                    data:
                      option: Outgoing
            default:
              - service: input_select.select_option
                target:
                  entity_id: input_select.mailbox_status
                data:
                  option: Empty
      - conditions:
          - condition: time
            after: '11:00:00'
          - condition: state
            entity_id: binary_sensor.mailbox_is_open
            state: 'on'
        sequence:
          - choose:
              - conditions:
                  - condition: or
                    conditions:
                      - condition: state
                        entity_id: input_select.mailbox_status
                        state: 'Incoming:New'
                      - condition: state
                        entity_id: input_select.mailbox_status
                        state: 'Incoming:Old'
                sequence:
                  - service: input_select.select_option
                    target:
                      entity_id: input_select.mailbox_status
                    data:
                      option: Empty
            default:
              - service: input_select.select_option
                target:
                  entity_id: input_select.mailbox_status
                data:
                  option: 'Incoming:New'
    default: []
  - condition: state
    entity_id: binary_sensor.mailbox_is_open
    state: 'on'
  - delay:
      hours: 0
      minutes: 1
      seconds: 0
      milliseconds: 0
  - service: input_select.select_option
    target:
      entity_id: input_select.mailbox_status
    data:
      option: Left Open
mode: restart

```
[/details]


*[u]You’ll also need to add this to your configuration.yaml:[/u]*
```
input_select:
  mailbox_status:
    name: Mailbox Status
    options:
      - Empty
      - Outgoing
      - Incoming:New
      - Incoming:Old
      - Left Open
    icon: mdi:mailbox
```
And if you want the box status indicated graphically in lovelace, custom:button-card works pretty well:
```
type: 'custom:button-card'
entity: input_select.mailbox_status
color_type: icon
color: var(--paper-item-icon-color)
state:
  - value: Empty
    icon: 'mdi:mailbox-outline'
  - value: Outgoing
    icon: 'mdi:mailbox-up-outline'
  - value: 'Incoming:New'
    icon: 'mdi:mail'
  - value: 'Incoming:Old'
    icon: 'mdi:mail'
  - value: Left Open
    icon: 'mdi:mailbox-open'
```
And of course you can then link these states with some of the other nice automations that send notifications on mail delivery or pickup, or link with mail-and-packages to include information on how many items might be in the mailbox.
