# NestJs Microservice By Turborepo

Building a Scalable NestJS Microservices Marketplace with Turborepo
This guide outlines the step-by-step process to develop a marketplace and eCommerce application using NestJS microservices within a Turborepo monorepo. The architecture includes microservices for Organization, User (Seller), and Store, with a scalable structure for future expansion.
Why Use Turborepo and NestJS?

Turborepo: Simplifies monorepo management with fast builds, dependency optimization, and task pipelines. It’s ideal for managing multiple microservices and shared libraries.
NestJS: Provides a modular, TypeScript-based framework with built-in microservices support, making it perfect for scalable backend systems.
Microservices: Enable independent development, deployment, and scaling of services like Organization, User, and Store.
RabbitMQ: Used as the message broker for asynchronous communication between microservices.

Step 1: Set Up the Turborepo Monorepo
1.1 Initialize the Turborepo Project

Install Node.js (v18 or later) and pnpm (recommended for Turborepo).
Create a new Turborepo project:npx create-turbo@latest marketplace-app
cd marketplace-app


Choose pnpm as the package manager when prompted.
The generated structure includes:
apps/ (for microservices)
packages/ (for shared libraries)
turbo.json (Turborepo configuration)
pnpm-workspace.yaml (workspace configuration)



1.2 Configure the Monorepo
Update pnpm-workspace.yaml to define workspaces for apps and packages:
packages:
  - 'apps/*'
  - 'packages/*'

Update turbo.json to define build, lint, and dev tasks:
{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "lint": {
      "outputs": []
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}

1.3 Install Global Dependencies
Install NestJS CLI and other dependencies:
pnpm add -g @nestjs/cli
pnpm add -D typescript @types/node

Step 2: Create Shared Libraries
Shared libraries reduce code duplication and ensure consistency across microservices. Create libraries for configuration, database models, and utilities.
2.1 Create a Shared Config Package

Create a new package:mkdir -p packages/config
cd packages/config
pnpm init


Update packages/config/package.json:{
  "name": "@marketplace/config",
  "version": "1.0.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "lint": "eslint . --ext .ts"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "eslint": "^8.0.0"
  }
}


Create packages/config/tsconfig.json:{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}


Create packages/config/src/index.ts for environment configuration:import { ConfigModule } from '@nestjs/config';

export const configModule = ConfigModule.forRoot({
  isGlobal: true,
  envFilePath: ['.env'],
});


Add a root tsconfig.json in the monorepo:{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "baseUrl": ".",
    "paths": {
      "@marketplace/*": ["packages/*/src"]
    }
  }
}



2.2 Create a Shared Database Package

Create packages/db:mkdir -p packages/db
cd packages/db
pnpm init


Update packages/db/package.json:{
  "name": "@marketplace/db",
  "version": "1.0.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "lint": "eslint . --ext .ts"
  },
  "dependencies": {
    "@nestjs/mongoose": "^10.0.0",
    "mongoose": "^8.0.0"
  }
}


Create packages/db/src/index.ts for Mongoose configuration:import { MongooseModule } from '@nestjs/mongoose';

export const mongooseModule = MongooseModule.forRootAsync({
  useFactory: () => ({
    uri: process.env.MONGO_URI || 'mongodb://localhost/marketplace',
  }),
});


Create shared Mongoose schemas (e.g., packages/db/src/schemas/organization.schema.ts):import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

@Schema()
export class Organization extends Document {
  @Prop({ required: true })
  name: string;

  @Prop()
  description: string;
}

export const OrganizationSchema = SchemaFactory.createForClass(Organization);



Step 3: Create Microservices
Each microservice (Organization, User, Store) will be a NestJS app in the apps/ directory, using RabbitMQ for communication.
3.1 Create the Organization Microservice

Generate the app:cd apps
nest new organization-service --skip-git --package-manager pnpm


Update apps/organization-service/package.json:{
  "name": "@marketplace/organization-service",
  "version": "1.0.0",
  "dependencies": {
    "@nestjs/core": "^10.0.0",
    "@nestjs/microservices": "^10.0.0",
    "@nestjs/mongoose": "^10.0.0",
    "@marketplace/config": "1.0.0",
    "@marketplace/db": "1.0.0",
    "amqp-connection-manager": "^4.0.0",
    "amqplib": "^0.10.0"
  }
}


Configure microservice in apps/organization-service/src/main.ts:import { NestFactory } from '@nestjs/core';
import { Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice(AppModule, {
    transport: Transport.RMQ,
    options: {
      urls: [process.env.RMQ_URL || 'amqp://localhost:5672'],
      queue: 'organization_queue',
      queueOptions: { durable: true },
    },
  });
  await app.listen();
}
bootstrap();


Create apps/organization-service/src/app.module.ts:import { Module } from '@nestjs/core';
import { configModule } from '@marketplace/config';
import { mongooseModule } from '@marketplace/db';
import { MongooseModule } from '@nestjs/mongoose';
import { Organization, OrganizationSchema } from '@marketplace/db';
import { OrganizationController } from './organization.controller';
import { OrganizationService } from './organization.service';

@Module({
  imports: [
    configModule,
    mongooseModule,
    MongooseModule.forFeature([{ name: Organization.name, schema: OrganizationSchema }]),
  ],
  controllers: [OrganizationController],
  providers: [OrganizationService],
})
export class AppModule {}


Implement basic CRUD in apps/organization-service/src/organization.controller.ts:import { Controller, Logger } from '@nestjs/core';
import { EventPattern, Payload } from '@nestjs/microservices';
import { OrganizationService } from './organization.service';
import { Organization } from '@marketplace/db';

@Controller()
export class OrganizationController {
  private readonly logger = new Logger(OrganizationController.name);

  constructor(private readonly organizationService: OrganizationService) {}

  @EventPattern('create_organization')
  async create(@Payload() data: { name: string; description?: string }) {
    this.logger.log(`Received create_organization event: ${JSON.stringify(data)}`);
    return this.organizationService.create(data);
  }
}


Implement apps/organization-service/src/organization.service.ts:import { Injectable } from '@nestjs/core';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { Organization } from '@marketplace/db';

@Injectable()
export class OrganizationService {
  constructor(@InjectModel(Organization.name) private organizationModel: Model<Organization>) {}

  async create(data: { name: string; description?: string }) {
    const organization = new this.organizationModel(data);
    return organization.save();
  }
}



3.2 Create the User (Seller) Microservice

Generate the app:cd apps
nest new user-service --skip-git --package-manager pnpm


Follow similar steps as the Organization service, updating package.json, main.ts, and app.module.ts.
Define a User schema in packages/db/src/schemas/user.schema.ts:import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

@Schema()
export class User extends Document {
  @Prop({ required: true })
  email: string;

  @Prop({ required: true })
  name: string;

  @Prop()
  organizationId: string;
}

export const UserSchema = SchemaFactory.createForClass(User);


Implement a controller to handle events like create_user and link users to organizations.

3.3 Create the Store Microservice

Generate the app:cd apps
nest new store-service --skip-git --package-manager pnpm


Define a Store schema in packages/db/src/schemas/store.schema.ts:import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

@Schema()
export class Store extends Document {
  @Prop({ required: true })
  name: string;

  @Prop({ required: true })
  ownerId: string;

  @Prop()
  description: string;
}

export const StoreSchema = SchemaFactory.createForClass(Store);


Implement store-related logic, linking stores to users.

Step 4: Set Up RabbitMQ

Install RabbitMQ locally or use a cloud provider (e.g., CloudAMQP).
Update .env in each microservice:RMQ_URL=amqp://localhost:5672
MONGO_URI=mongodb://localhost/marketplace


Ensure each microservice listens to its own queue (e.g., organization_queue, user_queue, store_queue).

Step 5: Create an API Gateway
The API Gateway exposes REST endpoints and routes requests to microservices via RabbitMQ.

Generate the gateway:cd apps
nest new api-gateway --skip-git --package-manager pnpm


Update apps/api-gateway/src/main.ts:import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();


Configure microservice client in apps/api-gateway/src/app.module.ts:import { Module } from '@nestjs/core';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { configModule } from '@marketplace/config';
import { AppController } from './app.controller';

@Module({
  imports: [
    configModule,
    ClientsModule.register([
      {
        name: 'ORGANIZATION_SERVICE',
        transport: Transport.RMQ,
        options: {
          urls: [process.env.RMQ_URL || 'amqp://localhost:5672'],
          queue: 'organization_queue',
          queueOptions: { durable: true },
        },
      },
      // Add clients for User and Store services
    ]),
  ],
  controllers: [AppController],
})
export class AppModule {}


Implement apps/api-gateway/src/app.controller.ts:import { Controller, Post, Body, Inject } from '@nestjs/core';
import { ClientRMQ } from '@nestjs/microservices';

@Controller('organizations')
export class AppController {
  constructor(@Inject('ORGANIZATION_SERVICE') private client: ClientRMQ) {}

  @Post()
  async createOrganization(@Body() data: { name: string; description?: string }) {
    return this.client.emit('create_organization', data);
  }
}



Step 6: Run and Test the Application

Start RabbitMQ and MongoDB (e.g., using Docker):docker run -d -p 5672:5672 rabbitmq:3-management
docker run -d -p 27017:27017 mongo


Run all services:pnpm --filter @marketplace/* dev


Test the API Gateway using a tool like Postman:
POST http://localhost:3000/organizations with body { "name": "Test Org" }.



Step 7: Scalability and Extensibility
7.1 Adding New Microservices
To add a new microservice (e.g., Product):

Create a new NestJS app in apps/product-service.
Define schemas in packages/db.
Configure RabbitMQ queue and event patterns.
Update the API Gateway to route requests.

7.2 Best Practices

Database per Service: Each microservice should have its own MongoDB database or collection to ensure loose coupling.
Event-Driven Communication: Use RabbitMQ events for inter-service communication to avoid tight coupling.
Shared Libraries: Centralize common logic (e.g., authentication, logging) in packages/.
CI/CD with Turborepo: Use Turborepo’s caching and pipelines for efficient builds in CI/CD systems like GitHub Actions.
Monitoring: Integrate tools like Prometheus and Grafana for monitoring microservices.

7.3 Deployment

Containerization: Use Docker to containerize each microservice.
Orchestration: Deploy with Kubernetes or AWS ECS for scalability.
Environment Management: Use tools like Doppler or AWS Secrets Manager for .env files.

Step 8: Future Enhancements

Authentication: Add a dedicated Auth microservice using JWT or OAuth.
Product Management: Implement a Product microservice for inventory.
Order Processing: Create an Order microservice for checkout and payment.
Search: Integrate Elasticsearch for product search.
Caching: Use Redis for caching frequently accessed data.

