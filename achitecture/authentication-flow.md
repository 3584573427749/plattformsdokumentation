# Auth-flöde (magic link)
1. Magic link genereras
2. Användaren klickar → får refresh token (httpOnly) + access token
3. Varje API-svar kan returnera nytt access token i header
4. Refresh används vid 401

# Auth-flöde (otp)
1. otp genereras 
2. Användaren skriver in koden → får refresh token (httpOnly) + access token
3. Varje API-svar kan returnera nytt access token i header
4. Refresh används vid 401
