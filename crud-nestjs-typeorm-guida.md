# Logica CRUD in NestJs con TypeORM

Vediamo passo passo come implementare la connessione a un database **postgres** e la gestione di operazioni CRUD  con **nestjs**.

Per prima cosa creiamo un nuovo progetto, spostandoci nella repo in cui vogliamo che sia

`cd  …/<la mia repo>`

`nest new <nome del mio prgetto>`

Una volta creato il progetto spostiamoci nella sua cartella e installiamo una libreria necessaria al nostro lavoro:  **TypeORM.**

`pnpm install --save @nestjs/typeorm typeorm pg` 

pg  sta per **Postgres**.

Installiamo inoltre i pacchetti per **`class-validator`** e **`class-trasformer`** che ci serviranno per le validazioni e i **DTO**:

`npm i --save class-validator class-transformer`

Importiamo `ValidationPipe`  in `main.ts` 

`main.ts` :

```tsx
import { NestFactory } from '@nestjs/core';
import { AppModule } from './productsApp.module';
import { ValidationPipe } from '@nestjs/common';        //<--------------------

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());             //<--------------------
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();

```

**TypeORM** è il framework **ORM** di **Typescript** e **Javascript**.

**ORM (Object Relational Mapping)** è una tecnica che perrmette di semplificare e rendere più flessibile l’interazione coi database relazionali.

Adesso ci collegheremo al db postgres per interagire con i dati di una tabella chiamata `employees`  (naturalmente adattate il nome della tabella e le variabili che verranno create  al vostro db)

Cominciamo importando il **TypeORM** module in **`app.module.ts`**

**`app.module.ts`:**

```jsx
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';     //<--------------------       
import { AppController } from './app.controller';
import { AppService } from './app.service';

@Module({
  imports: [
    TypeOrmModule.forRoot({                         //<--------------------
      type: 'postgres',
      host: process.env.DB_HOST,
      port: Number(process.env.DB_PORT),
      password: process.env.DB_PASSWORD,
      username: process.env.DB_USERNAME,
      database: process.env.DB_DATABASE,
      autoLoadEntities: true
    }),
   
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Per evitare di pubblicare le informazioni del db in chiaro, includeremo i dati necessari in un file `.env`  che sarà ignorato nei commit (`.git-ignore` è già configurato di default).

Creiamo nella repo del progetto il file `.env` e scriviamo:

DB_HOST=indirizzo host<br>
DB_PORT=numero porta<br>
DB_USERNAME=user<br>
DB_PASSWORD=password<br>
DB_DATABASE=nome db<br>

I nomi vanno scritti solamente in testo, senza simboli come   ”   o  ’

Installiamo la dependency per usare il file `.env` :

`pnpm install dotenv --save`

Dunque importiamola nel `main.ts`

`main.ts` :

```tsx
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common'; 
import { config } from 'dotenv';            //<--------------------

config();                                   //<--------------------

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

Creiamo adesso  la cartella `employee`  in cui inseriremo i file relativi all’entità  `employees` del db. 

Creiamo dentro `employee`  la cartella `dto`  per i relativi dto

L’albero delle cartelle avrà quindi questa forma (i file dentro employee verranno creati nel corso di questa guida):

── src<br>
│   ├── app.controller.spec.ts<br>
│   ├── app.controller.ts<br>
│   ├── app.module.ts<br>
│   ├── app.service.ts<br>
│   ├── employee<br>
│   │   ├── dto<br>
│   │   │   ├── create-employee.dto.ts<br>
│   │   │   └── update-employee.dto.ts<br>
│   │   ├── employee.controller.ts<br>
│   │   ├── employee.entity.ts<br>
│   │   ├── employee.module.ts<br>
│   │   └── employee.service.ts<br>
│   └── main.ts<br>

Creiamo dunque il file `employee.entity.ts` , `create-employee.dto.ts`  e `update-employee.dto.ts`

`employee.entity.ts` :

```tsx
import { PrimaryGeneratedColumn, Entity, Column } from "typeorm";

@Entity('employees') 
export class Employee {
    
    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    name: string;

    @Column()
    email: string;

    @Column()
    phone: string;

    @Column()
    department: string;
}
```

è importante specificare il nome esatto della tabella come parametro di `@Entity`

`create-employee.dto.ts` :

```tsx
import { IsNotEmpty, IsEmail, IsString  } from "class-validator";

export class CreateEmployeeDto {

    @IsNotEmpty()
    @IsString()
    name: string;
    
    @IsNotEmpty()
    @IsEmail()
    email: string;    
    
    @IsNotEmpty()
    @IsString()
    phone: string;
    
    @IsNotEmpty()
    @IsString()
    department: string;

}
```

`update-employee.dto.ts`:

```tsx
import { IsNotEmpty, IsEmail, IsString, IsOptional  } from "class-validator";

export class UpdateEmployeeDto {

    @IsOptional()
    @IsNotEmpty()
    @IsString()
    name: string;
    
    @IsOptional()
    @IsNotEmpty()
    @IsEmail()
    email: string;    
    
    @IsOptional()
    @IsNotEmpty()
    @IsString()
    phone: string;
    
    @IsOptional()
    @IsNotEmpty()
    @IsString()
    department: string;

}
```

Adesso creiamo il file `employee.service.ts`  in cui scriveremo il codice per la logica CRUD:

`employee.service.ts`  :

```tsx
import { Injectable, NotFoundException } from "@nestjs/common";
import { InjectRepository } from "@nestjs/typeorm";
import { DeleteResult, Repository } from "typeorm";
import { Employee } from "./employee.entity";
import { CreateEmployeeDto } from "./dto/create-employee.dto";

@Injectable()
export class EmployeeService {

    constructor( @InjectRepository(Employee)
                 private readonly repo: Repository<Employee> ) {}

    findAll(): Promise<Employee[]> {
    return this.repo.find();
    }

    async findOne(id: number): Promise<Employee> {
        const employee = await this.repo.findOneBy({ id });
        if (!employee) throw new NotFoundException('Employee not found');
        return employee;
    }

    create(dto: CreateEmployeeDto): Promise<Employee> {
        const employee = this.repo.create(dto);
        return this.repo.save(employee);
    }

    async update(id: number, dto: UpdateEmployeeDto): Promise<Employee> {
        const employee = await this.repo.findOneBy({id});
        if (!employee) throw new NotFoundException('Employee not found');
        return this.repo.save({id, ...dto});
    }

    async remove(id: number): Promise<DeleteResult> {
        const employee = await this.repo.findOneBy({id});
        if (!employee) throw new NotFoundException('Employee not found');
        return this.repo.delete(id);
  }
}
```

A questo punto creiamo il file per il controller `employee.controller.ts` 

`employee.controller.ts` :

```tsx
import { Controller, Get, Post, Param, Body, ParseIntPipe, Patch, Delete } from '@nestjs/common';
import { EmployeeService } from './employee.service';
import { CreateEmployeeDto } from './dto/create-employee.dto';
import { UpdateEmployeeDto } from './dto/update-employee.dto';

@Controller('employees')
export class EmployeeController {
  constructor(private readonly service: EmployeeService) {}

  @Get()
  findAll() {
    return this.service.findAll();
  }

  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.service.findOne(id);
  }

  @Post()
  create(@Body() dto: CreateEmployeeDto) {
    return this.service.create(dto);
  }

  @Patch(':id')
  update(@Param('id', ParseIntPipe) id: number, @Body() updateEmployeeDto: UpdateEmployeeDto) {
    return this.service.update(id, updateEmployeeDto);
  }

  @Delete(':id')
  remove( @Param('id', ParseIntPipe) id: number) {
    return this.service.remove(id);
  } 
}
```

Creiamo dunque il modulo `employee.module.ts` 

`employee.module.ts` :

```tsx
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { Employee } from './employee.entity'; 
import { EmployeeService } from './employee.service';  
import { EmployeeController } from './employee.controller';

@Module({
  imports: [TypeOrmModule.forFeature([Employee])],
  providers: [EmployeeService],
  controllers: [EmployeeController],
})
export class EmployeeModule {}
```

Aggiungiamo infine l’import di `EmployeeModule` in **`app.module.ts`**  

**`app.module.ts` :**

```tsx
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { TypeOrmModule } from '@nestjs/typeorm';
import { EmployeeModule } from './employee/employee.module';   // <-------------------------

@Module({
  imports: [
     TypeOrmModule.forRoot({
      type: 'postgres',
      host: process.env.DB_HOST,
      port: Number(process.env.DB_PORT),
      password: process.env.DB_PASSWORD,
      username: process.env.DB_USERNAME,
      database: process.env.DB_DATABASE,
      autoLoadEntities: true
    }),
    EmployeeModule,            // <-------------------------
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}

```

Ricordiamo di modificare nomi, variabili, percorsi coerentemente al nostro database e ai nostri file.

Adesso siamo pronti a testare il nostro servizio con HTTPie e provare le diverse richieste.

Ricordiamoci di avviare l’applicativo col comando:

`nest start --env-file .env --watch` 

Se non volessimo dover usare di volta in volta questo comando possiamo  modificare lo script `"start:dev"...` presente in `package.json`     

in 

`"start:dev": "nest start --watch --env-file .env”`
