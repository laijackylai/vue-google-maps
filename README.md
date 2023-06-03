# NUXT-google-maps

## Inital Setup

- run `git clone https://github.com/laijackylai/vue-google-maps.git` in the terminal
- Create a new file called `.env` inside the root directory
- Add a new entry called `GOOGLE_MAPS_API_KEY=<apiKey>`
- The `apiKey` will be attached in the email
- Run the build setup

## Run Setup

```bash
# install dependencies
$ yarn install

# serve with hot reload at localhost:3000
$ yarn dev

# build for production and launch server
$ yarn build
$ yarn start

# generate static project
$ yarn generate
```

## Remarks

- The apiKey are being restricted to three APIs
  - Geocoding API
  - Places API
  - Maps JavaScript API
- Each of the API are set to have quotas of the following:
  - Requests per day: 250
  - Requests per minute: 20
  - Requests per minute per user: 10
