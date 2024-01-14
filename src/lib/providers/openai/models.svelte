<script context="module" lang="ts">



import { getApiBase, getEndpointCompletions, getEndpointGenerations } from '../../ApiUtil.svelte'
import { countTokens } from '../../Models.svelte'
import { countMessageTokens } from '../../Stats.svelte'
import { globalStorage } from '../../Storage.svelte'
import type { Chat, Message, Model, ModelDetail } from '../../Types.svelte'
import { chatRequest, imageRequest } from './request.svelte'
import { checkModel } from './util.svelte'
import { encode } from 'gpt-tokenizer'
import { get } from 'svelte/store'

let modelsList = {
"data": [
{
"id": "nousresearch/nous-capybara-34b",
"name": "Nous: Capybara 34B",
"description": "This model is trained on the Yi-34B model for 3 epochs on the Capybara dataset. It's the first 34B Nous model and first 200K context length Nous model.\n\n**Note:** This endpoint currently supports 32k context.",
"pricing": {
"prompt": "0.0000007",
"completion": "0.0000028"
},
"context_length": 32768,
"architecture": {
"tokenizer": "Llama2",
"instruct_type": "vicuna"
},
"top_provider": {
"max_completion_tokens": null
},
"per_request_limits": {
"prompt_tokens": "5301093",
"completion_tokens": "1325273"
}
}
]
}




const hiddenSettings = {
      startSequence: true,
      stopSequence: true,
      aggressiveStop: true,
      delimiter: true,
      userMessageStart: true,
      userMessageEnd: true,
      assistantMessageStart: true,
      assistantMessageEnd: true,
      systemMessageStart: true,
      systemMessageEnd: true,
      repetitionPenalty: true,
      holdSocket: true
      // leadPrompt: true
} as any

const chatModelBase = {
  type: 'chat',
  help: 'Below are the settings that OpenAI allows to be changed for the API calls. See the <a target="_blank" href="https://platform.openai.com/docs/api-reference/chat/create">OpenAI API docs</a> for more details.',
  preFillMerge: (existingContent, newContent) => {
        // continuing assistant prompt. see if we need to add a space before we merge the new completion
        // there has to be a better way to do this
        if (existingContent && !newContent.match(/^('(t|ll|ve|m|d|re)[^a-z]|\s|[.,;:(_-{}*^%$#@!?+=~`[\]])/i)) {
          // add a trailing space if our new content isn't a contraction
          existingContent += ' '
        }
        return existingContent
  },
  request: chatRequest,
  check: checkModel,
  getTokens: (value) => encode(value),
  getEndpoint: (model) => get(globalStorage).openAICompletionEndpoint || (getApiBase() + getEndpointCompletions()),
  hideSetting: (chatId, setting) => !!hiddenSettings[setting.key],
  countMessageTokens: (message:Message, model:Model, chat: Chat) => {
        return countTokens(model, '## ' + message.role + ' ##:\r\n\r\n' + message.content + '\r\n\r\n\r\n')
  },
  countPromptTokens: (prompts:Message[], model:Model, chat: Chat):number => {
        // Not sure how OpenAI formats it, but this seems to get close to the right counts.
        // Would be nice to know. This works for gpt-3.5.  gpt-4 could be different.
        // Complete stab in the dark here -- update if you know where all the extra tokens really come from.
        return prompts.reduce((a, m) => {
          a += countMessageTokens(m, model, chat)
          return a
        }, 0) + 3 // Always seems to be message counts + 3
  }
} as ModelDetail



function fetchModelsList() {
  const request = new XMLHttpRequest();
  request.open('GET', 'https://openrouter.ai/api/v1/models', false); // false for synchronous request
  request.send(null);

  if (request.status === 200) {
    const data = JSON.parse(request.responseText);
    modelsList.data = data.data;
    console.log("fetched models list");
  } else {
    throw new Error(`HTTP error! status: ${request.status}`);
  }
}


function generateModels(modelsList: any): any {
  console.log("generateModels");
  fetchModelsList();
  const models: any = {};

  modelsList.data.forEach((model: any) => {
    models[model.id] = {
      ...chatModelBase,
      prompt: parseFloat(model.pricing.prompt),
      completion: parseFloat(model.pricing.completion),
      max: model.context_length
    };
  });

  return models;
}

export const chatModels : Record<string, ModelDetail> = generateModels(modelsList);



const imageModelBase = {
  type: 'image',
  prompt: 0.00,
  max: 1000, // 1000 char prompt, max
  request: imageRequest,
  check: checkModel,
  getTokens: (value) => [0],
  getEndpoint: (model) => getApiBase() + getEndpointGenerations(),
  hideSetting: (chatId, setting) => false
} as ModelDetail

export const imageModels : Record<string, ModelDetail> = {
      'dall-e-1024x1024': {
        ...imageModelBase,
        completion: 0.020, // $0.020 per image
        opt: {
          size: '1024x1024'
        }
      },
      'dall-e-512x512': {
        ...imageModelBase,
        completion: 0.018, // $0.018 per image
        opt: {
          size: '512x512'
        }
      },
      'dall-e-256x256': {
        ...imageModelBase,
        type: 'image',
        completion: 0.016, // $0.016 per image
        opt: {
          size: '256x256'
        }
      },
      'dall-e-3-1024x1024': {
        ...imageModelBase,
        type: 'image',
        completion: 0.04, // $0.040 per image
        opt: {
          model: 'dall-e-3',
          size: '1024x1024'
        }
      },
      'dall-e-3-1024x1792-Portrait': {
        ...imageModelBase,
        type: 'image',
        completion: 0.08, // $0.080 per image
        opt: {
          model: 'dall-e-3',
          size: '1024x1792'
        }
      },
      'dall-e-3-1792x1024-Landscape': {
        ...imageModelBase,
        type: 'image',
        completion: 0.08, // $0.080 per image
        opt: {
          model: 'dall-e-3',
          size: '1792x1024'
        }
      },
      'dall-e-3-1024x1024-HD': {
        ...imageModelBase,
        type: 'image',
        completion: 0.08, // $0.080 per image
        opt: {
          model: 'dall-e-3',
          size: '1024x1024',
          quality: 'hd'
        }
      },
      'dall-e-3-1024x1792-Portrait-HD': {
        ...imageModelBase,
        type: 'image',
        completion: 0.12, // $0.080 per image
        opt: {
          model: 'dall-e-3',
          size: '1024x1792',
          quality: 'hd'
        }
      },
      'dall-e-3-1792x1024-Landscape-HD': {
        ...imageModelBase,
        type: 'image',
        completion: 0.12, // $0.080 per image
        opt: {
          model: 'dall-e-3',
          size: '1792x1024',
          quality: 'hd'
        }
      }
}

</script>

