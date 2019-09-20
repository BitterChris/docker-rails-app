Just a repo to dump the working code after following [this article](https://medium.com/@dirkdk/running-a-rails-app-with-webpacker-and-docker-8d29153d3446).

Here are some things that I had to work through while following along that aren't entirely clear (or at least weren't for me).

#### Before you start
- It's not explicitly outlined but you'll need to copy the scripts in the `scripts/` directory of the app. So grab those after you generate your Rails app before going any further or you'll hit some walls quick.
  - `COPY` in the Dockerfile will inherit their permissions so you'll either need to add a line before any are called to make them executable or make sure they're executable locally before building the image (`chmod +x scripts/*`)
- You won't actually be able to run migrations on data locally by the normal `rails db:migreate`, it will all have to be executed in the db container.
  - You'll get a `PG:ConnectionError` if you try to do it locally. For subsequent migrations after the images are made you can run `docker-compose run web scripts/wait-for-it.sh db:5432 -- "rails db:migrate"` or if the containers are running you may be able to run `docker exec -it <web container name> "rails db:migrate"`, but I'm not sure on this last one.
- Because you won't see anything fail if you don't run a migration to get data into your app, before going further (after generating the Rails app) do a simple `rails generate model User` and `rails generate controller users index`. This will give your db enough to run the app.

#### Dockerfile notes
- Might need to use a different base image depending on what version of Ruby you want to use. I went with `ruby:2.6.4-stretch` since `apt-get` is Debian/Ubuntu based and `alpine` doesn't have access to it. You can use `alpine` images by replacing `apt-get` with `apk` (`install` becomes `add` and `update` stays the same). I chose to use a bigger image size than to actually make these adjustments.
- If you don't want to make the `scripts/` directory executable locally you can make them executable when the image is being built by adding `RUN ["chmod", "+x", "scripts/*"]` (this formatting covered in [Docker docs](https://docs.docker.com/engine/reference/builder/)).

#### docker-compose.yml notes
- Don't use `image: postgres:11` as indicated in the doc. Debian 10.10 which is what `2.6.4-stretch` is built off of doesn't support it and must use `postgres:10` - I think this was a typo in the article as the client library you pull down is `postgres-client-10`

#### database.yml notes
- Need to set the `host` and `password` in `database.yml` under `defaults`
  - `host: <%= ENV['DB_HOST'] %>`
  - `password: <%= ENV['DB_USER_PASSWORD'] %>`

#### webpacker.yml notes
- Setting `hmr: true` enables asset refreshing in development mode so you don't have to restart the server. More details can be found in the docs linked above that section in the `webpacker.yml` file.

#### config/environments/development.rb notes
- Add `config.webpacker.check_yarn_integrity = false` anywhere within the config block. Placement doesn't matter here.

#### General Notes
- Need to run `bundle install` after adding the `dotenv-rails` gem. Create a `.env` file in root following `.env.example` formatting.
- I ran through this until `First time run via docker-compose up, and javascript tag` section. Beyond it touches Production environments which I'm not going that far into. The app should run and be available from `localhost:3000` after running `docker-compose up` (and waiting for your console output to show that the Rails server is running as normal)
