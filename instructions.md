# Run the development database (and leave it running):
wasp db start

# Open new terminal window (or tab) in that same dir and continue in it.
wasp db migrate-dev

# Create initial dot env file from the template:
cp .env.server.example .env.server

# Last step: run the app!
wasp start

# start ngrok for payment test
ngrok http 3001

# db studio
wasp db studio