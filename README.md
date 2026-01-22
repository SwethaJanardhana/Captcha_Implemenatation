
nestjs-captcha-auth/
├── src/
│   ├── app.module.ts
│   ├── main.ts
│   ├── captcha/
│   │   ├── captcha.module.ts
│   │   └── captcha.service.ts
│   ├── auth/
│   │   ├── auth.controller.ts
│   │   ├── auth.module.ts
│   │   ├── auth.service.ts
│   │   ├── login-attempts.service.ts
│   │   └── dto/login.dto.ts
├── .env
├── package.json
└── README.md

## Install & Create Project
nest new nestjs-captcha-auth
cd nestjs-captcha-auth
npm install axios @nestjs/throttler

## .env
RECAPTCHA_SECRET_KEY=YOUR_SECRET_KEY
RECAPTCHA_SITE_KEY=YOUR_SITE_KEY

## main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableCors();
  await app.listen(3000);
}
bootstrap();

## app.module.ts
import { Module } from '@nestjs/common';
import { ThrottlerModule } from '@nestjs/throttler';
import { AuthModule } from './auth/auth.module';

@Module({
  imports: [
    ThrottlerModule.forRoot({
      ttl: 60,
      limit: 10,
    }),
    AuthModule,
  ],
})
export class AppModule {}

## CAPTCHA Module
### captcha/captcha.module.ts
import { Module } from '@nestjs/common';
import { CaptchaService } from './captcha.service';

@Module({
  providers: [CaptchaService],
  exports: [CaptchaService],
})
export class CaptchaModule {}

### captcha/captcha.service.ts
import { Injectable, BadRequestException } from '@nestjs/common';
import axios from 'axios';

@Injectable()
export class CaptchaService {
  async verify(token?: string) {
    if (!token) {
      throw new BadRequestException('Captcha token required');
    }

    const secret = process.env.RECAPTCHA_SECRET_KEY;

    const res = await axios.post(
      'https://www.google.com/recaptcha/api/siteverify',
      null,
      {
        params: {
          secret,
          response: token,
        },
      },
    );

    if (!res.data.success || res.data.score < 0.5) {
      throw new BadRequestException('Captcha verification failed');
    }

    return true;
  }
}

## Login DTO
### auth/dto/login.dto.ts
export class LoginDto {
  email: string;
  password: string;
  captchaToken?: string;
}

## Failed Login Tracker
### auth/login-attempts.service.ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class LoginAttemptsService {
  private attempts = new Map<string, number>();

  get(email: string) {
    return this.attempts.get(email) || 0;
  }

  increment(email: string) {
    const count = this.get(email) + 1;
    this.attempts.set(email, count);
    return count;
  }

  reset(email: string) {
    this.attempts.delete(email);
  }
}

## Auth Service (Captcha After 3 Failures)
### auth/auth.service.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { CaptchaService } from '../captcha/captcha.service';
import { LoginAttemptsService } from './login-attempts.service';

@Injectable()
export class AuthService {
  constructor(
    private captchaService: CaptchaService,
    private attemptsService: LoginAttemptsService,
  ) {}

  async login(email: string, password: string, captchaToken?: string) {
    const failures = this.attemptsService.get(email);

    if (failures >= 3) {
      await this.captchaService.verify(captchaToken);
    }

    // Replace with real DB check
    const isValid = password === '123456';

    if (!isValid) {
      this.attemptsService.increment(email);
      throw new UnauthorizedException('Invalid credentials');
    }

    this.attemptsService.reset(email);

    return {
      message: 'Login successful',
      accessToken: 'JWT_TOKEN_HERE',
    };
  }
}

## Auth Controller
### auth/auth.controller.ts
import { Controller, Post, Body } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LoginDto } from './dto/login.dto';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('login')
  async login(@Body() body: LoginDto) {
    return this.authService.login(
      body.email,
      body.password,
      body.captchaToken,
    );
  }
}

## Auth Module
### auth/auth.module.ts
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { CaptchaModule } from '../captcha/captcha.module';
import { LoginAttemptsService } from './login-attempts.service';

@Module({
  imports: [CaptchaModule],
  providers: [AuthService, LoginAttemptsService],
  controllers: [AuthController],
})
export class AuthModule {}


#### React Native CAPTCHA Screen (Ready)
import { WebView } from 'react-native-webview';

export default function CaptchaScreen({ onToken }) {
  return (
    <WebView
      source={{
        html: `
          <script src="https://www.google.com/recaptcha/api.js?render=YOUR_SITE_KEY"></script>
          <script>
            grecaptcha.ready(() => {
              grecaptcha.execute('YOUR_SITE_KEY', {action: 'login'})
              .then(token => window.ReactNativeWebView.postMessage(token));
            });
          </script>
        `,
      }}
      onMessage={(e) => onToken(e.nativeEvent.data)}
    />
  );
}




















