# yup-tricks

A collection of useful Yup tricks

## Validate YouTube Playlist ID via Fetch

To validate a YouTube Playlist ID (eg. `PLMZMRynGmhshJgxJqWVeaCfgpr3mWUqPm` from the URL `https://www.youtube.com/playlist?list=PLMZMRynGmhshJgxJqWVeaCfgpr3mWUqPm`), use [`string.matches()`](https://github.com/jquense/yup#stringmatchesregex-regex-message-string--function-schema) to check that it matches `/^[\w-]+$/` and [`Schema.test()`](https://github.com/jquense/yup#schematestname-string-message-string--function--any-test-function-schema) to make a `fetch()` request to the YouTube playlist URL with retries and an exponential delay:

```ts
import { setTimeout } from 'node:timers/promises';
import userAgents from 'top-user-agents';
import { string } from 'yup';

function getRandomUserAgent() {
  return userAgents[Math.floor(Math.random() * (userAgents.length - 1))]!;
}

export const youtubePlaylistsIdSchema = (
  // Pass in config instead of importing from a
  // packages/*/config.ts because database used by multiple
  // packages
  isYouTubeDisabled: boolean,
) =>
  string()
    .required()
    .matches(/^[\w-]+$/, {
      message:
        'YouTube Playlist ID does not only contain alphanumeric characters, hyphens, and underscores',
    })
    .test(
      'is-valid-youtube-playlist-id',
      "URL derived from YouTube Playlist ID doesn't respond with HTTP status code 200 (typo?)",
      async (youtubeId) => {
        if (isYouTubeDisabled) return true;

        let responseStatus;
        let delayMilliseconds = 750;

        // Exponential delays: 750ms, 1500ms, 3000ms
        const doubleDelay = (delay: number) => delay * 2;

        for (let attempt = 0; attempt < 3; attempt++) {
          if (attempt > 0) {
            await setTimeout(delayMilliseconds);
            delayMilliseconds = doubleDelay(delayMilliseconds);
          }

          try {
            responseStatus = (
              await fetch(
                `https://www.youtube.com/playlist?list=${youtubeId}`,
                {
                  headers: {
                    'User-Agent': getRandomUserAgent(),
                  },
                },
              )
            ).status;
          } catch {
            // Swallow errors fetching YouTube playlist URL
          }

          if (responseStatus === 200) break;
        }

        return responseStatus === 200;
      },
    );
```
