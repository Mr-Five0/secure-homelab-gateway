To craete an API key, follow these steps:
1. Log in to your Cloudflare account.
2. Navigate to the "My Profile" section.
![image](../img/profile.jpg)
3. Click on the "API Tokens" tab and Click on "Create Token".
![image](../img/api_token.jpg)
4. Select "Create Custom Token".
![image](../img/custom_token.jpg)
5. Configure the token with the following permissions:
- Under "Permissions", add the following `Zone:Firewall Services: Edit`
- Under "Zone Resources", select: Include:Specific zones: and choose your domain.
![image](../img/create_token.jpg)
6. Click "Continue to summary", review the settings, and then click "Create Token
![image](../img/token_confirm.jpg)
7. Copy the generated API token and store it securely. You will need this token to configure your Cloudflare Tunnel with Traefik.