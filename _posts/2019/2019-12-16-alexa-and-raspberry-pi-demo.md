---
layout: post
title: "Alexa and Raspberry Pi demo"
date: "2019-12-16"
categories: 
  - "alexa"
  - "iot"
  - "javascript"
  - "python"
  - "raspberry-pi"
tags: 
  - "alexa"
  - "iot"
  - "javascript"
  - "python"
  - "rpi"
coverImage: "119518.png"
---

We're keeping on playing with Alexa. This time I want to create one skill that uses a compatible device (for example one Raspberry Pi 3). Here you can see the documentation and examples that the people of Alexa [provides us](https://github.com/alexa/Alexa-Gadgets-Raspberry-Pi-Samples). Basically we need to create a new Product/Alexa gadget in the console. It gives us one amazonId and alexaGadgetSecret.

Then we need to install the SDK in our Raspberry Pi (it install several python libraries). The we can create our gadget script running on our Raspberry Pi, using the amazonId and alexaGadgetSecret. We can listen to several Alexa events. For example: When we say the wakeword, with a timer, an alarm and also we can create our custom events.

I'm going to create one skill that allows me to say something like: "Alexa, use device demo and set color to green" or "Alexa, use device demo and set color to red" and I'll put this color in the led matrix that I've got (one Raspberry Pi Sense Hat)

This is the python script:

```python
import logging
import sys
import json
from agt import AlexaGadget
from sense_hat import SenseHat
 
logging.basicConfig(stream=sys.stdout, level=logging.DEBUG)
logger = logging.getLogger(__name__)
 
sense = SenseHat()
sense.clear()
 
RED = [255, 0, 0]
GREEN = [0, 255, 0]
BLUE = [0, 0, 255]
YELLOW = [255, 255, 0]
BLACK = [0, 0, 0]
 
class Gadget(AlexaGadget):
    def on_alexa_gadget_statelistener_stateupdate(self, directive):
        for state in directive.payload.states:
            if state.name == 'wakeword':
                sense.clear()
                if state.value == 'active':
                    sense.show_message("Alexa", text_colour=BLUE)
 
    def set_color_to(self, color):
        sense.set_pixels(([color] * 64))
 
    def on_custom_gonzalo123_setcolor(self, directive):
        payload = json.loads(directive.payload.decode("utf-8"))
        color = payload['color']
 
        if color == 'red':
            logger.info('turn on RED display')
            self.set_color_to(RED)
 
        if color == 'green':
            logger.info('turn on GREEN display')
            self.set_color_to(GREEN)
 
        if color == 'yellow':
            logger.info('turn on YELLOW display')
            self.set_color_to(YELLOW)
 
        if color == 'black':
            logger.info('turn on YELLOW display')
            self.set_color_to(BLACK)
 
 
if __name__ == '__main__':
    Gadget().main()
```

And here the ini file of our gadget

```ini
[GadgetSettings]
amazonId = my_amazonId
alexaGadgetSecret = my_alexaGadgetSecret
 
[GadgetCapabilities]
Alexa.Gadget.StateListener = 1.0 - wakeword
Custom.gonzalo123 = 1.0
```

Whit this ini file I'm saying that my gadget will trigger a function when wakeword has been said and with my custom event "Custom.gonzalo123"

That's the skill:

```javascript
const Alexa = require('ask-sdk')
 
const RequestInterceptor = require('./interceptors/RequestInterceptor')
const ResponseInterceptor = require('./interceptors/ResponseInterceptor')
const LocalizationInterceptor = require('./interceptors/LocalizationInterceptor')
const GadgetInterceptor = require('./interceptors/GadgetInterceptor')
 
const LaunchRequestHandler = require('./handlers/LaunchRequestHandler')
const ColorHandler = require('./handlers/ColorHandler')
const HelpIntentHandler = require('./handlers/HelpIntentHandler')
const CancelAndStopIntentHandler = require('./handlers/CancelAndStopIntentHandler')
const SessionEndedRequestHandler = require('./handlers/SessionEndedRequestHandler')
const FallbackHandler = require('./handlers/FallbackHandler')
const ErrorHandler = require('./handlers/ErrorHandler')
 
let skill
 
module.exports.handler = async (event, context) => {
  if (!skill) {
    skill = Alexa.SkillBuilders.custom().
      addRequestInterceptors(
        RequestInterceptor,
        ResponseInterceptor,
        LocalizationInterceptor,
        GadgetInterceptor
      ).
      addRequestHandlers(
        LaunchRequestHandler,
        ColorHandler,
        HelpIntentHandler,
        CancelAndStopIntentHandler,
        SessionEndedRequestHandler,
        FallbackHandler).
      addErrorHandlers(
        ErrorHandler).
      create()
  }
 
  return await skill.invoke(event, context)
}
```

One important thing here is the GadgetInterceptor. This interceptor find the endpointId from the request and append it to the session. This endpoint exists because we've created our Product/Alexa gadget and also the python script is running on the device. If the endpoint isn't present maybe our skill must say to the user something like "No gadgets found. Please try again after connecting your gadget.".

```javascript
const utils = require('../lib/utils')
 
const GadgetInterceptor = {
  async process (handlerInput) {
    const endpointId = await utils.getEndpointIdFromConnectedEndpoints(handlerInput)
    if (endpointId) {
      utils.appendToSession(handlerInput, 'endpointId', endpointId)
    }
  }
}
 
module.exports = GadgetInterceptor
```

And finally the handlers to trigger the custom event:

```javascript
const log = require('../lib/log')
const utils = require('../lib/utils')
 
const buildLEDDirective = (endpointId, color) => {
  return {
    type: 'CustomInterfaceController.SendDirective',
    header: {
      name: 'setColor',
      namespace: 'Custom.gonzalo123'
    },
    endpoint: {
      endpointId: endpointId
    },
    payload: {
      color: color
    }
  }
}
 
const ColorHandler = {
  canHandle (handlerInput) {
    return handlerInput.requestEnvelope.request.type === 'IntentRequest'
      && handlerInput.requestEnvelope.request.intent.name === 'ColorIntent'
  },
  handle (handlerInput) {
    const requestAttributes = handlerInput.attributesManager.getRequestAttributes()
    const cardTitle = requestAttributes.t('SKILL_NAME')
 
    const endpointId = utils.getEndpointIdFromSession(handlerInput)
    if (!endpointId) {
      log.error('endpoint', error)
      return handlerInput.responseBuilder.
        speak(error).
        getResponse()
    }
 
    const color = handlerInput.requestEnvelope.request.intent.slots['color'].value
    log.info('color', color)
    log.info('endpointId', endpointId)
 
    return handlerInput.responseBuilder.
      speak(`Ok. The selected color is ${color}`).
      withSimpleCard(cardTitle, color).
      addDirective(buildLEDDirective(endpointId, color)).
      getResponse()
  }
}
 
module.exports = ColorHandler
```

Here one video with the working example:

[![youtube](https://img.youtube.com/vi/1dVQDsk4kWo/0.jpg)](https://www.youtube.com/watch?v=1dVQDsk4kWo)

Source code in my [github](https://github.com/gonzalo123/alexa_rpi)
