# Debug Guide: Capturing Registration Service Error Details

## Step 1: Add Enhanced Error Logging

Replace the failing code section in your registration service with detailed error capture:

```javascript
// In /app/packages/api/src/auth/services/registration.service.js
// Around line 133

async mergePropertiesFromSocialLogin(user, profile) {
    console.log('=== MERGE SOCIAL LOGIN DEBUG START ===');
    console.log('User ID:', user?.id);
    console.log('User before merge:', JSON.stringify(user, null, 2));
    console.log('Profile data:', JSON.stringify(profile, null, 2));
    
    user.displayName = user.displayName ?? profile.displayName ?? null;
    user.twitterId = user.twitterId ?? profile.twitterId;
    user.googleId = user.googleId ?? profile.googleId;
    user.discordId = user.discordId ?? profile.discordId;
    user.gateId = profile.gateId ?? user.gateId;
    user.gateVerified = profile.gateVerified ?? user.gateVerified;
    
    console.log('User after merge:', JSON.stringify(user, null, 2));
    
    try {
        console.log('Attempting to save user...');
        const savedUser = await this.userRepository.save(user);
        console.log('User saved successfully:', savedUser.id);
        console.log('=== MERGE SOCIAL LOGIN DEBUG END ===');
        return savedUser;
    } catch (error) {
        console.error('=== USER SAVE ERROR DETAILS ===');
        console.error('Error Name:', error.name);
        console.error('Error Message:', error.message);
        console.error('Error Code:', error.code);
        console.error('Error Detail:', error.detail);
        console.error('Error Constraint:', error.constraint);
        console.error('Error Column:', error.column);
        console.error('Error Table:', error.table);
        console.error('Error Query:', error.query);
        console.error('Full Error Object:', JSON.stringify(error, Object.getOwnPropertyNames(error), 2));
        console.error('Error Stack:', error.stack);
        console.error('=== END ERROR DETAILS ===');
        
        // Re-throw the error to maintain existing behavior
        throw error;
    }
}
```

## Step 2: Enable Database Query Logging

Add to your TypeORM configuration to see the actual SQL queries:

```javascript
// In your database configuration file
{
    type: 'postgres', // or your database type
    // ... other config
    logging: ['error', 'query'],
    logger: 'advanced-console',
    // This will show you the exact SQL queries being executed
}
```

## Step 3: Reproduce the Error

### Method A: Manual Testing
1. Trigger a social login flow that would call this method
2. Check your console output for the detailed error logs

### Method B: API Testing
If you have API endpoints, test with curl:

```bash
# Example API call that might trigger the error
curl -X POST http://localhost:3000/auth/social-login \
  -H "Content-Type: application/json" \
  -d '{
    "provider": "twitter",
    "accessToken": "your-token-here"
  }'
```

### Method C: Unit Test
Create a test to reproduce the issue:

```javascript
// Create a test file: registration.service.test.js
describe('RegistrationService', () => {
    it('should capture merge error details', async () => {
        const service = new RegistrationService();
        
        // Mock user data that might cause the error
        const user = {
            id: 1,
            email: 'test@example.com',
            twitterId: 'existing-twitter-id'
        };
        
        const profile = {
            twitterId: 'conflicting-twitter-id',
            displayName: 'Test User'
        };
        
        try {
            await service.mergePropertiesFromSocialLogin(user, profile);
        } catch (error) {
            console.log('Captured error in test:', error);
            // The error details will be logged by our enhanced logging
        }
    });
});
```

## Step 4: Check Application Logs

### If using PM2:
```bash
pm2 logs your-app-name --lines 100
```

### If using Docker:
```bash
docker logs -f your-container-name
```

### If running directly:
Check your console where you started the Node.js process

## Step 5: Check Database Logs

### PostgreSQL:
```bash
# Check PostgreSQL logs
tail -f /var/log/postgresql/postgresql-*.log

# Or if using Docker:
docker logs -f your-postgres-container
```

### MySQL:
```bash
# Check MySQL error log
tail -f /var/log/mysql/error.log
```

## Step 6: Database Investigation Queries

Run these queries to check for potential conflicts:

```sql
-- Check for duplicate social IDs
SELECT twitter_id, COUNT(*) as count 
FROM users 
WHERE twitter_id IS NOT NULL 
GROUP BY twitter_id 
HAVING COUNT(*) > 1;

SELECT google_id, COUNT(*) as count 
FROM users 
WHERE google_id IS NOT NULL 
GROUP BY google_id 
HAVING COUNT(*) > 1;

SELECT discord_id, COUNT(*) as count 
FROM users 
WHERE discord_id IS NOT NULL 
GROUP BY discord_id 
HAVING COUNT(*) > 1;

-- Check table constraints
SELECT conname, contype, pg_get_constraintdef(oid) 
FROM pg_constraint 
WHERE conrelid = 'users'::regclass;
```

## Step 7: Monitor in Real-Time

Add a temporary middleware to log all database operations:

```javascript
// Add to your app.js or main server file
app.use((req, res, next) => {
    if (req.path.includes('/auth/')) {
        console.log(`Auth request: ${req.method} ${req.path}`);
        console.log('Request body:', JSON.stringify(req.body, null, 2));
    }
    next();
});
```

## Expected Error Types to Look For

Based on the code, watch for these specific error patterns:

### 1. Unique Constraint Violations:
```
Error: duplicate key value violates unique constraint "users_twitter_id_key"
Detail: Key (twitter_id)=(12345) already exists.
```

### 2. Foreign Key Violations:
```
Error: insert or update on table "users" violates foreign key constraint
```

### 3. NOT NULL Violations:
```
Error: null value in column "email" violates not-null constraint
```

### 4. TypeORM Validation Errors:
```
QueryFailedError: [specific database error]
```

## Next Steps

Once you capture the error details:

1. **Copy the complete error message** from your logs
2. **Note which social provider** was being used
3. **Check if it's reproducible** with the same user/profile data
4. **Share the error details** so we can provide a targeted fix

This approach will give us the exact information needed to solve the issue!