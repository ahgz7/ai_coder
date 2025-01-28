# Rules

## 1. Folder Structure

```bash
.

├── api/                      # API request/response interfaces
├── backend
│   └── src
│       ├── config/           # Configuration files (database, environment variables, etc.)
│       ├── domain/           # Domain interfaces and types
│       ├── handlers/         # Express route handlers
│       ├── services/         # Business logic (service layer)
│       ├── repositories/     # Database query logic (data access layer)
│       │   └── entities/     # Database entity definitions
│       ├── middlewares/      # Custom middleware (authentication, logging, etc.)
│       ├── integration_tests/ # End-to-end testing of critical user journeys
│       └── utils/            # Pure utility functions
└── frontend
    └── src
        ├── components/       # Vanilla React components
        ├── containers/       # Connected components with shared functionality
        ├── pages/            # Application pages/routes
        ├── services/         # Backend API integration
        ├── state/            # Global state management (Redux)
        └── utils/            # Frontend utility functions
```

## 2. Description of Each Folder with Dependency Injection

### `api/`

API interfaces define the contract for requests and responses. Theis shared and used by both the frontend and backend.

```typescript
// backend/src/api/user.ts
export interface CreateUserRequest {
  name: string;
  email: string;
}

export interface UserResponse {
  id: string;
  name: string;
  email: string;
}
```

### Backend

#### `config/`

Configuration is handled through environment variables and type-safe config objects.

```typescript
// backend/src/config/database_config.ts
export interface DatabaseConfig {
  host: string;
  port: number;
  username: string;
  password: string;
  database: string;
}

export const getDatabaseConfig = (): DatabaseConfig => ({
  host: process.env.DB_HOST || 'localhost',
  port: Number(process.env.DB_PORT) || 5432,
  username: process.env.DB_USERNAME || 'postgres',
  password: process.env.DB_PASSWORD || '',
  database: process.env.DB_NAME || 'development'
});

// backend/src/config/app_config.ts
export interface AppConfig {
  port: number;
  env: 'development' | 'production' | 'test';
  apiVersion: string;
}

export const getAppConfig = (): AppConfig => ({
  port: Number(process.env.PORT) || 3000,
  env: (process.env.NODE_ENV as AppConfig['env']) || 'development',
  apiVersion: process.env.API_VERSION || 'v1'
});
```

#### `domain/`

Domain interfaces represent the core business objects, free from implementation details.

```typescript
// backend/src/domain/user.ts
export interface User {
  id: string;
  name: string;
  email: string;
}
```

#### `handlers/`

Handlers convert API requests to domain operations.

```typescript
// backend/src/handlers/user_handler.ts
import { Request, Response } from 'express';
import { UserService } from '../services/user_service';
import { CreateUserRequest, UserResponse } from '../api/user';

export class UserHandler {
  constructor(private userService: UserService) {}

  async getUser(req: Request, res: Response): Promise<void> {
    const user = await this.userService.getUserById(req.params.id);
    const response: UserResponse = user;
    res.json(response);
  }
}
```

#### `backend/services/`

Services implement business logic and depend on repositories other services.

```typescript
// backend/src/services/user_service.ts
import { UserRepository } from '../repositories/user_repository';
import { User } from '../domain/user';

export class UserService {
  constructor(private readonly userRepository: UserRepository) {}

  async getUserById(userId: string): Promise<User | null> {
    return await this.userRepository.findById(userId);
  }
}
```

#### `repositories/`

Repositories handle data access using TypeORM.

```typescript
// backend/src/repositories/user_repository.ts
import { Repository } from 'typeorm';
import { UserEntity } from './entities/user_entity';
import { User } from '../domain/user';

export class UserRepository {
  constructor(private readonly repository: Repository<UserEntity>) {}

  async findById(id: string): Promise<User | null> {
    return await this.repository.findOne({ where: { id } });
  }
}
```

#### `repositories/entities`

Database entities implement the domain interfaces.

```typescript
// backend/src/repositories/entities/user_entity.ts
import { Entity, PrimaryGeneratedColumn, Column } from 'typeorm';
import { User } from '../../domain/user';

@Entity()
export class UserEntity implements User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  name: string;

  @Column({ unique: true })
  email: string;
}
```

#### `middlewares/`

Middlewares like authentication can be injected with service dependencies.

```typescript
// backend/src/middlewares/auth_middleware.ts
import { Request, Response, NextFunction } from 'express';
import { AuthService } from '../services/auth_service';

export class AuthMiddleware {
  constructor(private readonly authService: AuthService) {}

  handle(req: Request, res: Response, next: NextFunction) {
    // Authentication logic using this.authService
    next();
  }
}
```

#### `integration_tests/`

End-to-end testing of critical user journeys using supertest.

```typescript
// backend/src/integration_tests/auth_test.ts
import request from 'supertest';
import { app } from '../app';
import { container } from '../container';

describe('Authentication Flow', () => {
  beforeAll(async () => {
    // Setup test database and dependencies
    await container.get('database').migrate();
  });

  afterAll(async () => {
    // Cleanup test data
    await container.get('database').cleanup();
  });

  it('should successfully register and login a user', async () => {
    // Register
    const registerResponse = await request(app)
      .post('/api/auth/register')
      .send({
        email: 'test@example.com',
        password: 'securePassword123'
      });
    expect(registerResponse.status).toBe(201);

    // Login
    const loginResponse = await request(app)
      .post('/api/auth/login')
      .send({
        email: 'test@example.com',
        password: 'securePassword123'
      });
    expect(loginResponse.status).toBe(200);
    expect(loginResponse.body).toHaveProperty('token');
  });

  it('should protect authenticated routes', async () => {
    const response = await request(app)
      .get('/api/user/profile')
      .set('Authorization', 'Bearer invalid-token');
    expect(response.status).toBe(401);
  });
});
```

#### `backend/utils/`

Pure utility functions without side effects.

```typescript
// backend/src/utils/validation.ts
export const isValidEmail = (email: string): boolean => {
  const EMAIL_REGEX = /\S+@\S+\.\S+/;
  return EMAIL_REGEX.test(email);
};
```

#### `container.ts`

The main advantage of DI is that we can configure the injection in a central place.

```typescript
// backend/src/container.ts
import { DataSource } from 'typeorm';
import { User } from './domain/user';
import { UserService } from './services/user_service';
import { UserRepository } from './repositories/user_repository';
import { UserHandler } from './handlers/user_handler';
import { AuthMiddleware } from './middlewares/auth_middleware';
import { AuthService } from './services/auth_service';

export interface Dependencies {
  dataSource: DataSource;
  authService: AuthService;
}

export interface Container {
  userHandler: UserHandler;
  authMiddleware: AuthMiddleware;
}

export const createContainer = async (deps: Dependencies): Promise<Container> => {
  // Create repositories
  const userRepository = new UserRepository(deps.dataSource.getRepository(User));

  // Create services
  const userService = new UserService(userRepository);

  // Create handlers
  const userHandler = new UserHandler(userService);

  // Create middlewares
  const authMiddleware = new AuthMiddleware(deps.authService);

  return {
    userHandler,
    authMiddleware,
  };
};

// backend/src/container.prod.ts
import { DataSource } from 'typeorm';
import { Dependencies } from './container';
import { AuthService } from './services/auth_service';
import { User } from './domain/user';

export const createProdDependencies = async (): Promise<Dependencies> => {
  const dataSource = new DataSource({
    type: 'postgres',
    host: process.env.DB_HOST,
    port: Number(process.env.DB_PORT),
    username: process.env.DB_USERNAME,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
    entities: [User],
    synchronize: process.env.NODE_ENV !== 'production',
  });

  await dataSource.initialize();

  const authService = new AuthService(/* auth dependencies */);

  return {
    dataSource,
    authService,
  };
};

// backend/src/container.test.ts
import { DataSource } from 'typeorm';
import { Dependencies } from './container';
import { AuthService } from './services/auth_service';
import { User } from './domain/user';

export const createTestDependencies = async (): Promise<Dependencies> => {
  const dataSource = new DataSource({
    type: 'sqlite',
    database: ':memory:',
    entities: [User],
    synchronize: true,
    dropSchema: true,
  });

  await dataSource.initialize();

  const authService = {
    validateToken: jest.fn().mockResolvedValue(true),
    generateToken: jest.fn().mockResolvedValue('test-token'),
  } as jest.Mocked<AuthService>;

  return {
    dataSource,
    authService,
  };
};

// Usage in app.ts
import { createContainer } from './container';
import { createProdDependencies } from './container.prod';

const bootstrap = async () => {
  const dependencies = await createProdDependencies();
  const container = await createContainer(dependencies);
  
  const app = express();
  app.use(express.json());
  app.use(container.authMiddleware.handle);
  
  const router = Router();
  router.get('/users/:id', container.userHandler.getUser);
  
  app.use('/api', router);
  
  return app;
};

// Usage in tests
import { createContainer } from './container';
import { createTestDependencies } from './container.test';

describe('Integration Tests', () => {
  let container: Container;
  
  beforeEach(async () => {
    const dependencies = await createTestDependencies();
    container = await createContainer(dependencies);
  });

  it('should get user by id', async () => {
    const response = await request(app)
      .get('/api/users/123')
      .expect(200);
    
    expect(response.body).toBeDefined();
  });
});
```

#### `app.ts`

In `app.ts`, we inject dependencies when initializing the application.

```typescript
// backend/src/app.ts
import { createContainer } from './container';
import { createProdDependencies } from './container.prod';

const bootstrap = async () => {
  const dependencies = await createProdDependencies();
  const container = await createContainer(dependencies);
  
  const app = express();
  app.use(express.json());
  app.use(container.authMiddleware.handle);
  
  const router = Router();
  router.get('/users/:id', container.userHandler.getUser);
  
  app.use('/api', router);
  
  return app;
};
```

#### `server.ts`

In `server.ts`, we start the server.

```typescript
// backend/src/server.ts
import app from './app';
import { getConfig } from './config';

app.listen(getConfig().app.port, () => {
  console.log(`Server running on port ${getConfig().app.port}`);
});
```

### Frontend

#### `components/`

Vanilla React components.

```typescript
// frontend/src/components/user.tsx
import React from 'react';

export const UserComponent = () => {
  return <div>User Component</div>;
};
```

#### `containers/`

Connected components with shared functionality and complex state management.

```typescript
// frontend/src/containers/user_profile/user_profile.tsx
import React from 'react';
import { useSelector, useDispatch } from 'react-redux';
import { ProfileDetails } from '../../components/profile_details';
import { fetchUserProfile } from '../../state/user.actions';

export const UserProfile = () => {
  const dispatch = useDispatch();
  const user = useSelector(state => state.user);

  useEffect(() => {
    dispatch(fetchUserProfile());
  }, [dispatch]);

  return <ProfileDetails user={user} />;
};

// frontend/src/containers/user_profile/user_profile.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import { Provider } from 'react-redux';
import { configureStore } from '@reduxjs/toolkit';
import { UserProfile } from './user_profile';

describe('UserProfile Container', () => {
  it('should match snapshot', () => {
    const store = configureStore({
      reducer: {
        user: () => ({ name: 'Test User' })
      }
    });
    
    const { container } = render(
      <Provider store={store}>
        <UserProfile />
      </Provider>
    );
    
    expect(container).toMatchSnapshot();
  });

  it('should fetch and display user profile', async () => {
    const mockStore = configureStore({
      reducer: {
        user: (state = null, action) => 
          action.type === 'SET_USER' ? action.payload : state
      }
    });

    render(
      <Provider store={mockStore}>
        <UserProfile />
      </Provider>
    );

    await waitFor(() => {
      expect(screen.getByText('Test User')).toBeInTheDocument();
    });
  });
});
```

#### `pages/`

Application pages/routes that compose containers and components. Pages should include integration tests and snapshots to ensure proper component composition and data flow.

```typescript
// frontend/src/pages/dashboard/dashboard.tsx
import React from 'react';
import { UserProfile } from '../../containers/user_profile';
import { ActivityFeed } from '../../containers/activity_feed';

export const Dashboard = () => (
  <div className="dashboard">
    <UserProfile />
    <ActivityFeed />
  </div>
);

// frontend/src/pages/dashboard/dashboard.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import { Provider } from 'react-redux';
import { configureStore } from '@reduxjs/toolkit';
import { Dashboard } from './dashboard';

describe('Dashboard Page', () => {
  const setupStore = (initialState = {}) => {
    return configureStore({
      reducer: {
        user: () => initialState.user || null,
        activities: () => initialState.activities || []
      }
    });
  };

  it('should match snapshot', () => {
    const store = setupStore({
      user: { name: 'Test User' },
      activities: [{ id: 1, text: 'Test Activity' }]
    });

    const { container } = render(
      <Provider store={store}>
        <Dashboard />
      </Provider>
    );

    expect(container).toMatchSnapshot();
  });

  it('should integrate user profile and activity feed', async () => {
    const store = setupStore({
      user: { name: 'Test User' },
      activities: [
        { id: 1, text: 'Recent Activity 1' },
        { id: 2, text: 'Recent Activity 2' }
      ]
    });

    render(
      <Provider store={store}>
        <Dashboard />
      </Provider>
    );

    // Test complete page integration
    await waitFor(() => {
      expect(screen.getByText('Test User')).toBeInTheDocument();
      expect(screen.getByText('Recent Activity 1')).toBeInTheDocument();
      expect(screen.getByText('Recent Activity 2')).toBeInTheDocument();
    });
  });
});
```

#### `frontend/services/`

Backend API integration.

```typescript
// frontend/src/services/user_service.ts
import axios from 'axios';

export const UserService = {
  async getUserById(userId: string): Promise<any> {
    return await axios.get(`/api/users/${userId}`);
  }
};
```

#### `state/`

Global state management (Redux).

```typescript
// frontend/src/state/user.reducer.ts
import { createStore, combineReducers } from 'redux';

export const userReducer = (state = {}, action: any) => {
  switch (action.type) {
    case 'GET_USER':
      return action.payload;
    default:
      return state;
  }
};

export const store = createStore(combineReducers({ user: userReducer }));
```

#### `frontend/utils/`

Frontend utility functions.

```typescript
// frontend/src/utils/validation.ts
export const isValidEmail = (email: string): boolean => {
  const EMAIL_REGEX = /\S+@\S+\.\S+/;
  return EMAIL_REGEX.test(email);
};
```

## Design Principles

This project follows a set of strict design principles for both implementation and testing to ensure code quality, maintainability, and testability.

### Implementation Principles

1. **Domain-Driven Design**
   - Start with domain objects and relationships
   - Define domain objects as TypeScript interfaces
   - Document all domain objects thoroughly
   - Domain objects should be internal, not exposed to API

2. **Clean Architecture**
   - Backend: Repository → Service → Handler
   - Frontend: Components → Containers/Pages → State → Services
   - Each layer depends only on previous or same layer
   - Clear separation of concerns between layers

3. **Code Organization**
   - File names must be snake_case
   - Test files must be located next to implementation files and end with _test.ts
   - No index.js files in folders
   - No global variables
   - No magic constants
   - No console.log (except in index.js)

4. **Dependency Management**
   - Use composition over inheritance
   - Implement dependency injection
   - No object initialization inside constructors
   - No environment-specific code, use dependency injection instead

5. **Function Design**
   - Implement pure functions where possible
   - Document all public methods and exports
   - Test all public and exported methods

### Testing Principles

1. **Test Organization**
   - Tests organized in three blocks: Arrange, Act, Assert
   - Use Jest as testing framework
   - Test files co-located with implementation

2. **Test Focus**
   - Test behavior, not implementation
   - Test what code does, not how it does it
   - Use real objects over mocks when possible
   - Mock only external libraries with network calls or expensive operations

3. **Integration Testing**
   - Backend: Use supertest for API integration tests
   - Frontend: Pages and containers serve as integration test boundaries
   - Use snapshot testing for UI components
   - Use in-memory database for backend tests

### Layer-Specific Principles

1. **Backend**
   - Repositories hide ORM implementation
   - Services implement business logic
   - Handlers only orchestrate services
   - Environment variables centralized in config
   - Use TypeORM for database access
   - Use random UUIDs for unique IDs

2. **Frontend**
   - Components: Pure UI, no business logic
   - Containers: State management and shared functionality
   - Pages: Route-level components and layout
   - Services: API integration
   - Use Radix UI for UI components
   - Use React and Redux for state management

### API Design

1. **Interface Definition**
   - Define request/response interfaces in api folder
   - Document all API interfaces
   - Don't expose domain objects in API
   - Map domain objects to API interfaces

2. **Configuration Management**
   - Use .env for environment variables
   - Centralize config in dedicated files
   - Group related configs (e.g., auth settings)

### Example Implementation

#### Backend Example Implementation

```typescript
// Domain Interface
interface User {
  id: string;
  email: string;
}

// API Interface
interface UserResponse {
  id: string;
  email: string;
}

// Repository Layer
class UserRepository {
  constructor(private dataSource: DataSource) {}
  async findById(id: string): Promise<User> {
    const entity = await this.dataSource.findOne(UserEntity, { where: { id } });
    return this.mapToUser(entity);
  }
}

// Service Layer
class UserService {
  constructor(private userRepo: UserRepository) {}
  async getUser(id: string): Promise<User> {
    return this.userRepo.findById(id);
  }
}

// Handler Layer
class UserHandler {
  constructor(private userService: UserService) {}
  async getUser(req: Request, res: Response) {
    const user = await this.userService.getUser(req.params.id);
    res.json(this.mapToResponse(user));
  }
}
```

#### Frontend Example Implementation

```typescript
// Component Layer (Pure UI)
// frontend/src/components/user_profile/user_profile.tsx
interface UserProfileProps {
  user: User;
  onUpdate: (user: User) => void;
}

export const UserProfile: React.FC<UserProfileProps> = ({ user, onUpdate }) => {
  return (
    <div className="user-profile">
      <Input value={user.email} onChange={e => onUpdate({ ...user, email: e.target.value })} />
    </div>
  );
};

// Container Layer (Connected Component)
// frontend/src/containers/user_profile/user_profile_container.tsx
import { useSelector, useDispatch } from 'react-redux';
import { updateUser } from '../../state/user.actions';
import { UserProfile } from '../../components/user_profile/user_profile';

export const UserProfileContainer: React.FC = () => {
  const dispatch = useDispatch();
  const user = useSelector(state => state.user.data);
  
  const handleUpdate = (updatedUser: User) => {
    dispatch(updateUser(updatedUser));
  };

  return <UserProfile user={user} onUpdate={handleUpdate} />;
};

// Page Layer (Route Component)
// frontend/src/pages/profile/profile_page.tsx
import { UserProfileContainer } from '../../containers/user_profile/user_profile_container';
import { ActivityFeedContainer } from '../../containers/activity_feed/activity_feed_container';

export const ProfilePage: React.FC = () => {
  return (
    <div className="profile-page">
      <h1>User Profile</h1>
      <UserProfileContainer />
      <ActivityFeedContainer />
    </div>
  );
};

// State Layer (Redux)
// frontend/src/state/user.slice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface UserState {
  data: User | null;
  loading: boolean;
  error: string | null;
}

const userSlice = createSlice({
  name: 'user',
  initialState: { data: null, loading: false, error: null } as UserState,
  reducers: {
    setUser: (state, action: PayloadAction<User>) => {
      state.data = action.payload;
    },
    setLoading: (state, action: PayloadAction<boolean>) => {
      state.loading = action.payload;
    },
    setError: (state, action: PayloadAction<string>) => {
      state.error = action.payload;
    }
  }
});

// Service Layer (API Integration)
// frontend/src/services/user_service.ts
import axios from 'axios';

export class UserService {
  private baseUrl = '/api/users';

  async getUser(id: string): Promise<User> {
    const response = await axios.get<UserResponse>(`${this.baseUrl}/${id}`);
    return this.mapToUser(response.data);
  }

  async updateUser(user: User): Promise<User> {
    const response = await axios.put<UserResponse>(`${this.baseUrl}/${user.id}`, user);
    return this.mapToUser(response.data);
  }

  private mapToUser(response: UserResponse): User {
    return {
      id: response.id,
      email: response.email
    };
  }
}

// Integration Test Example
// frontend/src/pages/profile/profile_page.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import { Provider } from 'react-redux';
import { configureStore } from '@reduxjs/toolkit';
import { ProfilePage } from './profile_page';

describe('ProfilePage', () => {
  const setupStore = (initialState = {}) => {
    return configureStore({
      reducer: {
        user: () => initialState.user || { data: null, loading: false, error: null }
      }
    });
  };

  it('should render user profile and activity feed', async () => {
    const store = setupStore({
      user: {
        data: { id: '1', email: 'test@example.com' },
        loading: false,
        error: null
      }
    });

    render(
      <Provider store={store}>
        <ProfilePage />
      </Provider>
    );

    // Snapshot test
    expect(container).toMatchSnapshot();

    // Integration test
    await waitFor(() => {
      expect(screen.getByText('User Profile')).toBeInTheDocument();
      expect(screen.getByDisplayValue('test@example.com')).toBeInTheDocument();
    });
  });
});
```
