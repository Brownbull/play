Created new Wasp app in ./khujta_opss directory!

To run your new app, follow the instructions below:

  1. Position into app's root directory:
    cd khujta_opss/app

  2. Run the development database (and leave it running):
    wasp db start

  3. Open new terminal window (or tab) in that same dir and continue in it.

  4. Apply initial database migrations:
    wasp db migrate-dev

  5. Create initial dot env file from the template:
    cp .env.server.example .env.server

  6. Last step: run the app!
    wasp start

Check the README for additional guidance and the link to docs!