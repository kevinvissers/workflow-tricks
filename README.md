# workflow-tricks
Set of GitHub workflow tip and tricks


# Secrets

## SOLUTION
```yaml
name: Super solution
on: workflow_dispatch

jobs:
  get-available-secrets:
    runs-on: ubuntu-latest
    name: List available secrets for PRO
    environment: PRO
    steps:
      - uses: actions/checkout@v4
      
      - name: Make all secrets available as env vars
        run: |
          echo '${{ toJson(secrets) }}' | jq -r 'to_entries[] | "\(.key)=\(.value)"' >> $GITHUB_ENV
      
      - name: Your build step
        run: |
          # All secrets are now automatically available as environment variables
          echo "MYSECRET: $MYSECRET"
          echo "ANOTHER_SECRET: $ANOTHER_SECRET"
```

## Options
### 1. All in reusable worklow
- Fetch list of available secrets
- Spawn matrix jobs for each secret and store as build artifact
- Collect all artifacts and make them available as environment variables

Good:
- Possible to add any numer of secrets with any name
- Users don't need to know how secrets are handled

Bad:
- Matrix job is causing a lot of overhead / extra cost
- Secrets are stored as artifacts!!!


### 2. User input
- Create job in calling workflow which collects the needed secrets
- Do something with the secrets so GitHub doesn't recognize them as a secret anymore
- Pass secrets to central workflow via job inputs/outputs

Good:
- User decide which secrets are need

Bad:
- Users need to know how secrets are handled
- Workaround needed to pass the secrets around (since this is not allowed by GitHub)
- Extra calling worklow is needed to fetch and pass the secrets

### 3. Fixed secrets
- Foresee e.g. 10 GitHub secrets with a fixed name (SECRET_1, SECRET_2, ...)
- While setting a secret the user need to map it to a 'slot' (access-token=SECRET_1, private-key=SECRET_2)
  - The value of the secret will be stored in SECRET_1, SECRET_2, ...
- Keep a mapping file where you state wich secret is stored in which 'slot'
- In the central workflow; read all foreseen slots
- Create an action which creates an environment variable based on the mapping file

Pseudo code:
```yaml
- name: Make secrets available
  uses: actions/github-script@v8
  env:
    SECRET_1: ${{ secrets.SECRET_1 }}
    SECRET_2: ${{ secrets.SECRET_2 }}
    SECRET_3: ${{ secrets.SECRET_3 }}
    SECRET_X: ${{ secrets.SECRET_X }}
  with:
    - script: |
        const secret1 = process.env.SECRET_1;
        const secret2 = process.env.SECRET_2;
        const secret3 = process.env.SECRET_3;
        const secretX = process.env.SECRET_X;

        const mappingFile = fs.ReadFileSync("mapping_file");
        const mappings = JSON.parse(mappingFile);
        mappings.forEach((mapping) => {
          if(mapping.slot === "SECRET_1") {
            core.setEnvironmentVariable(mapping.name, secret1);
            return;
          }
          if(mapping.slot === "SECRET_2") {
            core.setEnvironmentVariable(mapping.name, secret2);
            return;
          }
          if(mapping.slot === "SECRET_3") {
            core.setEnvironmentVariable(mapping.name, secret3);
            return;
          }
          if(mapping.slot === "SECRET_X") {
            core.setEnvironmentVariable(mapping.name, secretX);
            return;
          }
        });
```

Good:
- Simple
- No additional workflows needed

Bad:
- Limited to x-number of secrets
- Users need to remember which secret is stored in which slot (to prevent accidental overwrites)
- Mapping file needed to create the correct environment variables
