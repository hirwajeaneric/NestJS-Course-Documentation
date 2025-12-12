# Module 10: Background Jobs & Queues

## 📚 Table of Contents
1. [Overview](#overview)
2. [Introduction to Queues](#introduction-to-queues)
3. [Bull Queue Setup](#bull-queue-setup)
4. [Creating and Processing Jobs](#creating-and-processing-jobs)
5. [Job Scheduling](#job-scheduling)
6. [Job Priorities and Delays](#job-priorities-and-delays)
7. [Job Events and Monitoring](#job-events-and-monitoring)
8. [Error Handling in Queues](#error-handling-in-queues)
9. [Email Processing](#email-processing)
10. [File Processing in Background](#file-processing-in-background)
11. [Best Practices](#best-practices)
12. [Mini Project: Email Notification System](#mini-project-email-notification-system)

---

## Overview

Background jobs allow you to process time-consuming tasks asynchronously without blocking your API requests. This module covers implementing background job processing using Bull Queue (Redis-based) and NestJS's built-in scheduling capabilities.

**Why Use Background Jobs?**
- **Non-blocking**: API responses remain fast while heavy tasks run in background
- **Reliability**: Failed jobs can be retried automatically
- **Scalability**: Process jobs across multiple workers/instances
- **Monitoring**: Track job progress, failures, and completion
- **Scheduling**: Run jobs at specific times or intervals

**Common Use Cases:**
- Sending emails (welcome emails, notifications)
- Image/video processing
- Report generation
- Data synchronization
- Bulk operations
- Scheduled tasks (cleanups, backups)

---

## Introduction to Queues

### What is a Queue?

A queue is a data structure that holds jobs (tasks) in order and processes them one by one. Think of it like a line at a bank - customers (jobs) wait in line and are served (processed) in order.

**Queue Components:**
- **Producer**: Creates and adds jobs to the queue
- **Queue**: Stores jobs waiting to be processed
- **Consumer/Worker**: Processes jobs from the queue
- **Redis**: Acts as the message broker (stores the queue)

**Benefits:**
- **Decoupling**: API doesn't wait for job completion
- **Retry Logic**: Failed jobs can be automatically retried
- **Priority**: Important jobs can be processed first
- **Rate Limiting**: Control how many jobs process simultaneously

---

## Bull Queue Setup

### Install Dependencies

Bull is a Redis-based queue system for Node.js. It requires Redis to be running.

```bash
npm install @nestjs/bull bull
npm install @nestjs/schedule @nestjs/platform-express
```

**Why Bull?**
- Built specifically for Node.js
- Redis-backed (fast and reliable)
- Rich feature set (priorities, delays, rate limiting)
- Excellent NestJS integration
- Active maintenance and community

### Configure Bull Module

This setup connects Bull to Redis and registers queue processors. The `forRootAsync` pattern allows reading Redis configuration from environment variables.

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    
    // BullModule.forRootAsync() configures the connection to Redis
    // This is a global configuration that applies to all queues
    BullModule.forRootAsync({
      imports: [ConfigModule],
      
      // useFactory creates the Redis connection configuration
      // Bull uses Redis to store job data and coordinate workers
      useFactory: async (configService: ConfigService) => ({
        redis: {
          // Redis server hostname
          // In production, use a dedicated Redis instance
          host: configService.get('REDIS_HOST', 'localhost'),
          
          // Redis server port (default is 6379)
          port: configService.get('REDIS_PORT', 6379),
          
          // Optional: Redis password for authentication
          // password: configService.get('REDIS_PASSWORD'),
        },
        
        // Prefix for all Redis keys (prevents conflicts)
        // All Bull keys will start with 'bull:' in Redis
        prefix: 'bull',
      }),
      
      inject: [ConfigService],
    }),
    
    // Register specific queues
    // Each queue has a unique name and can process different job types
    BullModule.registerQueue({
      name: 'email', // Queue name - use descriptive names
    }),
    
    BullModule.registerQueue({
      name: 'file-processing',
    }),
    
    BullModule.registerQueue({
      name: 'notifications',
    }),
  ],
})
export class AppModule {}
```

**Important Configuration Notes:**
- `prefix`: Helps organize keys if multiple applications use the same Redis instance
- Each queue needs to be registered separately
- Queue names are case-sensitive and must match when injecting
- In production, always use Redis authentication and secure connections

### Environment Variables

Add these to your `.env` file:

```env
# Redis Configuration
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=  # Optional, only if Redis requires authentication
```

---

## Creating and Processing Jobs

### Creating Jobs (Producer)

The producer is typically your service that needs to offload work. Instead of doing the work immediately, you add a job to the queue.

**When to Create Jobs:**
- Operations that take more than a few seconds
- Tasks that can fail and need retry logic
- Work that doesn't need immediate completion
- Operations that should be rate-limited

```typescript
// src/email/email.service.ts
import { Injectable } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bull';
import { Queue } from 'bull';

@Injectable()
export class EmailService {
  // Inject the queue using @InjectQueue decorator
  // The queue name must match what was registered in the module
  constructor(
    @InjectQueue('email') private emailQueue: Queue,
  ) {}

  // sendWelcomeEmail: Instead of sending email immediately,
  // we add a job to the queue and return immediately
  async sendWelcomeEmail(userEmail: string, userName: string): Promise<void> {
    // add() creates a new job and adds it to the queue
    // The first parameter is the job name (optional but recommended)
    // The second parameter is the job data (what the processor will receive)
    await this.emailQueue.add('welcome-email', {
      email: userEmail,
      name: userName,
      timestamp: new Date(),
    });
    
    // This method returns immediately - the email is sent asynchronously
    // The client doesn't wait for the email to be sent
    console.log(`Welcome email job added to queue for ${userEmail}`);
  }

  // sendPasswordResetEmail: Another job type in the same queue
  async sendPasswordResetEmail(userEmail: string, resetToken: string): Promise<void> {
    await this.emailQueue.add('password-reset-email', {
      email: userEmail,
      token: resetToken,
      expiresAt: new Date(Date.now() + 3600000), // 1 hour from now
    });
  }

  // sendBulkEmails: Multiple jobs can be added efficiently
  async sendBulkEmails(recipients: Array<{ email: string; name: string }>): Promise<void> {
    // Create multiple jobs at once
    const jobs = recipients.map(recipient =>
      this.emailQueue.add('newsletter', {
        email: recipient.email,
        name: recipient.name,
      })
    );
    
    // Wait for all jobs to be added
    await Promise.all(jobs);
    console.log(`Added ${recipients.length} newsletter jobs to queue`);
  }
}
```

**Understanding Job Creation:**
- `add()` is asynchronous but returns quickly (just adds to queue, doesn't process)
- Job data should be serializable (JSON-friendly)
- Large data should be stored elsewhere (database, S3) with references in job data
- Job names help organize different job types in the same queue

### Processing Jobs (Consumer/Processor)

The processor handles jobs from the queue. It's registered as a provider and automatically receives jobs.

```typescript
// src/email/email.processor.ts
import { Processor, Process } from '@nestjs/bull';
import { Job } from 'bull';
import { Injectable } from '@nestjs/common';

// @Processor('email') decorator marks this class as a processor for the 'email' queue
// The queue name must match the registered queue name
@Processor('email')
@Injectable()
export class EmailProcessor {
  // @Process() decorator registers a method to handle specific job types
  // The parameter is the job name (optional - omit to handle all jobs in queue)
  @Process('welcome-email')
  async handleWelcomeEmail(job: Job) {
    // job.data contains the data passed when creating the job
    const { email, name } = job.data;
    
    console.log(`Processing welcome email for ${email}`);
    
    // job.progress() updates the job's progress (0-100)
    // Useful for long-running jobs to track completion
    await job.progress(25);
    
    // Simulate email sending (replace with actual email service)
    // In production, use a service like SendGrid, AWS SES, or Nodemailer
    await this.sendEmail(email, `Welcome, ${name}!`, 'Welcome to our platform!');
    
    await job.progress(100);
    
    // The return value is stored with the job (useful for results)
    return { sent: true, recipient: email };
  }

  @Process('password-reset-email')
  async handlePasswordResetEmail(job: Job) {
    const { email, token, expiresAt } = job.data;
    
    const resetLink = `https://yourapp.com/reset-password?token=${token}`;
    const subject = 'Password Reset Request';
    const body = `
      You requested a password reset.
      Click here to reset your password: ${resetLink}
      This link expires at ${expiresAt}.
    `;
    
    await this.sendEmail(email, subject, body);
    return { sent: true };
  }

  @Process('newsletter')
  async handleNewsletter(job: Job) {
    const { email, name } = job.data;
    
    // Update progress for monitoring
    await job.progress(50);
    
    await this.sendEmail(email, 'Monthly Newsletter', `Hi ${name}, here's our newsletter...`);
    
    await job.progress(100);
    return { sent: true };
  }

  // Private helper method (would use real email service in production)
  private async sendEmail(to: string, subject: string, body: string): Promise<void> {
    // Simulate email sending delay
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    console.log(`Email sent to ${to}: ${subject}`);
    
    // In production, replace with:
    // await this.emailService.send({ to, subject, body });
  }
}
```

**Processor Lifecycle:**
1. Job arrives in queue
2. Bull assigns job to available processor
3. `@Process()` method is called with the job
4. Processor does the work
5. Job is marked as completed (or failed)
6. Result is stored (if returned)

**Key Points:**
- Processors run automatically when jobs are added
- Multiple processors can run concurrently (scales horizontally)
- Jobs are processed in order (unless priorities are used)
- Each `@Process()` handler should be idempotent (safe to retry)

### Register Processor in Module

The processor must be registered as a provider in the module.

```typescript
// src/email/email.module.ts
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';
import { EmailService } from './email.service';
import { EmailProcessor } from './email.processor';

@Module({
  imports: [
    // Register the queue this module uses
    // Must match the queue name used in @Processor() and @InjectQueue()
    BullModule.registerQueue({
      name: 'email',
    }),
  ],
  providers: [
    EmailService,
    EmailProcessor, // Processor must be a provider
  ],
  exports: [EmailService],
})
export class EmailModule {}
```

---

## Job Scheduling

### Using @nestjs/schedule

NestJS provides a built-in scheduler for running jobs at specific times or intervals using cron-like syntax.

**Install Dependencies:**
```bash
npm install @nestjs/schedule
```

### Schedule Module Setup

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [
    // ScheduleModule.forRoot() enables the scheduler
    // This allows using @Cron() and @Interval() decorators
    ScheduleModule.forRoot(),
  ],
})
export class AppModule {}
```

### Cron Jobs

Cron jobs run at specific times using cron expressions. Cron syntax: `* * * * * *`
- Second (0-59)
- Minute (0-59)
- Hour (0-23)
- Day of month (1-31)
- Month (1-12)
- Day of week (0-7, where 0 and 7 are Sunday)

```typescript
// src/tasks/scheduled-tasks.service.ts
import { Injectable } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import { SchedulerRegistry } from '@nestjs/schedule';

@Injectable()
export class ScheduledTasksService {
  constructor(private schedulerRegistry: SchedulerRegistry) {}

  // @Cron() decorator runs a method on a schedule
  // CronExpression provides common schedules
  
  // Run every day at midnight
  @Cron(CronExpression.EVERY_DAY_AT_MIDNIGHT)
  handleDailyCleanup() {
    console.log('Running daily cleanup...');
    // Clean up old records, generate reports, etc.
  }

  // Run every hour
  @Cron(CronExpression.EVERY_HOUR)
  handleHourlyTask() {
    console.log('Running hourly task...');
  }

  // Run every 30 seconds (for testing)
  @Cron('*/30 * * * * *')
  handleEvery30Seconds() {
    console.log('This runs every 30 seconds');
  }

  // Run every Monday at 9 AM
  @Cron('0 9 * * 1')
  handleWeeklyReport() {
    console.log('Generating weekly report...');
  }

  // Run at a specific time daily (3:30 AM)
  @Cron('0 30 3 * * *')
  handleDailyBackup() {
    console.log('Running daily backup...');
  }
}
```

**Common Cron Expressions:**
- `CronExpression.EVERY_SECOND` - Every second (use carefully!)
- `CronExpression.EVERY_MINUTE` - Every minute
- `CronExpression.EVERY_HOUR` - Every hour
- `CronExpression.EVERY_DAY_AT_MIDNIGHT` - Daily at midnight
- `CronExpression.EVERY_WEEK` - Weekly
- `'0 0 * * *'` - Daily at midnight (standard cron format)

### Interval Jobs

Interval jobs run at fixed intervals (every X milliseconds).

```typescript
// src/tasks/interval-tasks.service.ts
import { Injectable } from '@nestjs/common';
import { Interval } from '@nestjs/schedule';

@Injectable()
export class IntervalTasksService {
  // @Interval() runs a method at fixed time intervals
  // Parameter is milliseconds between executions
  
  // Run every 10 seconds
  @Interval(10000)
  handleEvery10Seconds() {
    console.log('This runs every 10 seconds');
  }

  // Run every 5 minutes
  @Interval(300000)
  handleEvery5Minutes() {
    console.log('Running cleanup every 5 minutes...');
  }
}
```

### Timeout Jobs

Run a job once after a delay.

```typescript
// src/tasks/timeout-tasks.service.ts
import { Injectable } from '@nestjs/common';
import { Timeout } from '@nestjs/schedule';

@Injectable()
export class TimeoutTasksService {
  // @Timeout() runs a method once after a delay
  // Parameter is milliseconds to wait before execution
  
  // Run once after 5 seconds (when app starts)
  @Timeout(5000)
  handleOnceAfter5Seconds() {
    console.log('This runs once 5 seconds after app startup');
  }
}
```

### Dynamic Scheduling

You can also create schedules programmatically.

```typescript
// src/tasks/dynamic-scheduler.service.ts
import { Injectable } from '@nestjs/common';
import { SchedulerRegistry } from '@nestjs/schedule';
import { CronJob } from 'cron';

@Injectable()
export class DynamicSchedulerService {
  constructor(private schedulerRegistry: SchedulerRegistry) {}

  addCronJob(name: string, cronExpression: string, callback: () => void) {
    // Create a cron job
    const job = new CronJob(cronExpression, callback);

    // Add to scheduler registry
    this.schedulerRegistry.addCronJob(name, job);

    // Start the job
    job.start();

    console.log(`Job ${name} added with cron: ${cronExpression}`);
  }

  deleteCronJob(name: string) {
    const job = this.schedulerRegistry.getCronJob(name);
    job.stop();
    this.schedulerRegistry.deleteCronJob(name);
    console.log(`Job ${name} deleted`);
  }
}
```

---

## Job Priorities and Delays

### Job Priorities

Higher priority jobs are processed before lower priority jobs.

```typescript
// src/email/email.service.ts
async sendUrgentEmail(email: string, message: string): Promise<void> {
  // Priority: higher number = higher priority (1-100 range typical)
  // Urgent emails are processed before regular emails
  await this.emailQueue.add(
    'urgent-email',
    { email, message },
    {
      priority: 10, // High priority (processed first)
    }
  );
}

async sendRegularEmail(email: string, message: string): Promise<void> {
  await this.emailQueue.add(
    'regular-email',
    { email, message },
    {
      priority: 1, // Low priority (processed after urgent emails)
    }
  );
}
```

**Priority Guidelines:**
- Higher numbers = higher priority
- Default priority is usually 0
- Use priorities for critical vs. non-critical jobs
- Too many high-priority jobs defeats the purpose

### Job Delays

Delay job processing for a specific amount of time.

```typescript
async sendReminderEmail(email: string, delayMinutes: number): Promise<void> {
  await this.emailQueue.add(
    'reminder-email',
    { email },
    {
      // delay: milliseconds to wait before processing
      // Useful for reminders, scheduled notifications
      delay: delayMinutes * 60 * 1000,
    }
  );
}

// Example: Send reminder in 24 hours
await this.sendReminderEmail('user@example.com', 24 * 60);
```

### Job Attempts and Backoff

Configure retry logic for failed jobs.

```typescript
async sendEmailWithRetry(email: string, message: string): Promise<void> {
  await this.emailQueue.add(
    'email-with-retry',
    { email, message },
    {
      // attempts: number of times to retry if job fails
      attempts: 3,
      
      // backoff: retry strategy
      backoff: {
        type: 'exponential', // 'fixed' or 'exponential'
        delay: 2000, // Initial delay in milliseconds
      },
      // Exponential backoff: 2s, 4s, 8s (doubles each retry)
      // Fixed backoff: 2s, 2s, 2s (same delay each retry)
    }
  );
}
```

**Backoff Strategies:**
- **exponential**: Delay doubles each retry (2s, 4s, 8s, 16s...)
- **fixed**: Same delay for each retry (2s, 2s, 2s...)
- Exponential is better for transient failures (network issues)
- Fixed is better for predictable retry scenarios

---

## Job Events and Monitoring

### Job Progress

Update job progress for monitoring and user feedback.

```typescript
@Process('process-large-file')
async handleLargeFile(job: Job) {
  const { fileId } = job.data;
  
  // Update progress as work progresses
  await job.progress(0);
  // ... do initial work ...
  
  await job.progress(25);
  // ... do more work ...
  
  await job.progress(50);
  // ... continue processing ...
  
  await job.progress(75);
  // ... final steps ...
  
  await job.progress(100);
  
  return { fileId, processed: true };
}
```

### Job Events

Listen to queue events for monitoring and logging.

```typescript
// src/email/email.processor.ts
import { OnQueueActive, OnQueueCompleted, OnQueueFailed } from '@nestjs/bull';

@Processor('email')
@Injectable()
export class EmailProcessor {
  // @OnQueueActive() fires when a job starts processing
  @OnQueueActive()
  onActive(job: Job) {
    console.log(`Processing job ${job.id} of type ${job.name}`);
    console.log(`Data:`, job.data);
  }

  // @OnQueueCompleted() fires when a job completes successfully
  @OnQueueCompleted()
  onCompleted(job: Job, result: any) {
    console.log(`Job ${job.id} completed with result:`, result);
    
    // Log metrics, send notifications, etc.
    this.logMetrics('email-sent', job.data);
  }

  // @OnQueueFailed() fires when a job fails
  @OnQueueFailed()
  onFailed(job: Job, error: Error) {
    console.error(`Job ${job.id} failed:`, error.message);
    
    // Send alerts, log errors, notify administrators
    this.sendAlert('job-failed', { jobId: job.id, error: error.message });
  }

  // Additional event handlers
  @OnQueueError()
  onError(error: Error) {
    console.error('Queue error:', error);
  }

  @OnQueueWaiting()
  onWaiting(jobId: string | number) {
    console.log(`Job ${jobId} is waiting in queue`);
  }
}
```

**Event Types:**
- `active`: Job started processing
- `completed`: Job finished successfully
- `failed`: Job failed (after all retries)
- `error`: Queue/system error
- `waiting`: Job added to queue
- `stalled`: Job processing took too long

---

## Error Handling in Queues

### Retry Logic

Jobs automatically retry on failure if configured.

```typescript
@Process('unreliable-operation')
async handleUnreliableOperation(job: Job) {
  try {
    // Operation that might fail (API call, network request, etc.)
    await this.externalAPICall(job.data);
    
    return { success: true };
  } catch (error) {
    // Throw error to trigger retry mechanism
    // Bull will automatically retry based on job configuration
    throw new Error(`Operation failed: ${error.message}`);
  }
}
```

### Manual Job Retry

Manually retry a specific job.

```typescript
async retryFailedJob(jobId: string): Promise<void> {
  const job = await this.emailQueue.getJob(jobId);
  
  if (job && job.failedReason) {
    // Retry the job
    await job.retry();
    console.log(`Retrying job ${jobId}`);
  }
}
```

### Error Handling Best Practices

```typescript
@Process('robust-operation')
async handleRobustOperation(job: Job) {
  try {
    const result = await this.performOperation(job.data);
    return result;
  } catch (error) {
    // Log error details for debugging
    console.error('Job failed:', {
      jobId: job.id,
      jobName: job.name,
      data: job.data,
      error: error.message,
      stack: error.stack,
    });

    // Update job with error information
    await job.update({ ...job.data, error: error.message });

    // Re-throw to trigger retry (if configured)
    // Or handle gracefully and don't throw
    throw error;
  }
}
```

---

## Email Processing

### Complete Email Service with Queue

Here's a complete implementation of an email service using queues.

```typescript
// src/email/email.service.ts
import { Injectable } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bull';
import { Queue } from 'bull';

@Injectable()
export class EmailService {
  constructor(
    @InjectQueue('email') private emailQueue: Queue,
  ) {}

  // Welcome email: Sent immediately when user registers
  async sendWelcomeEmail(userEmail: string, userName: string): Promise<void> {
    await this.emailQueue.add('welcome-email', {
      email: userEmail,
      name: userName,
      type: 'welcome',
    });
  }

  // Password reset: Urgent, high priority
  async sendPasswordResetEmail(userEmail: string, token: string): Promise<void> {
    await this.emailQueue.add('password-reset', {
      email: userEmail,
      token,
      type: 'password-reset',
    }, {
      priority: 10, // High priority
      attempts: 5, // Retry 5 times if fails
      backoff: {
        type: 'exponential',
        delay: 2000,
      },
    });
  }

  // Newsletter: Low priority, can be delayed
  async sendNewsletterEmails(recipients: string[]): Promise<void> {
    const jobs = recipients.map(email =>
      this.emailQueue.add('newsletter', {
        email,
        type: 'newsletter',
      }, {
        priority: 1, // Low priority
        delay: 10000, // Delay 10 seconds
      })
    );
    
    await Promise.all(jobs);
  }
}
```

### Email Processor with Real Email Service

```typescript
// src/email/email.processor.ts
import { Processor, Process, OnQueueFailed } from '@nestjs/bull';
import { Job } from 'bull';
import { Injectable, Logger } from '@nestjs/common';
import * as nodemailer from 'nodemailer';

@Processor('email')
@Injectable()
export class EmailProcessor {
  private readonly logger = new Logger(EmailProcessor.name);
  private transporter: nodemailer.Transporter;

  constructor() {
    // Configure nodemailer (use your email service credentials)
    this.transporter = nodemailer.createTransport({
      host: process.env.SMTP_HOST,
      port: parseInt(process.env.SMTP_PORT),
      secure: false, // true for 465, false for other ports
      auth: {
        user: process.env.SMTP_USER,
        pass: process.env.SMTP_PASSWORD,
      },
    });
  }

  @Process('welcome-email')
  async handleWelcomeEmail(job: Job) {
    const { email, name } = job.data;
    
    await job.progress(50);
    
    await this.transporter.sendMail({
      from: '"Our App" <noreply@ourapp.com>',
      to: email,
      subject: 'Welcome to Our Platform!',
      html: `
        <h1>Welcome, ${name}!</h1>
        <p>Thank you for joining us.</p>
      `,
    });
    
    await job.progress(100);
    this.logger.log(`Welcome email sent to ${email}`);
  }

  @Process('password-reset')
  async handlePasswordReset(job: Job) {
    const { email, token } = job.data;
    
    const resetLink = `${process.env.FRONTEND_URL}/reset-password?token=${token}`;
    
    await this.transporter.sendMail({
      from: '"Our App" <noreply@ourapp.com>',
      to: email,
      subject: 'Reset Your Password',
      html: `
        <h1>Password Reset Request</h1>
        <p>Click <a href="${resetLink}">here</a> to reset your password.</p>
        <p>This link expires in 1 hour.</p>
      `,
    });
    
    this.logger.log(`Password reset email sent to ${email}`);
  }

  @OnQueueFailed()
  handleFailedJob(job: Job, error: Error) {
    this.logger.error(
      `Email job ${job.id} failed for ${job.data.email}: ${error.message}`,
    );
    // Send alert, log to error tracking service, etc.
  }
}
```

---

## File Processing in Background

### Image Processing Queue

Process large images asynchronously.

```typescript
// src/files/file.service.ts
import { Injectable } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bull';
import { Queue } from 'bull';

@Injectable()
export class FileService {
  constructor(
    @InjectQueue('file-processing') private fileQueue: Queue,
  ) {}

  async processImage(fileId: string, filePath: string): Promise<void> {
    // Add image processing job to queue
    await this.fileQueue.add('process-image', {
      fileId,
      filePath,
      operations: ['resize', 'compress', 'generate-thumbnail'],
    }, {
      attempts: 3,
      backoff: {
        type: 'exponential',
        delay: 5000,
      },
    });
  }

  async processVideo(fileId: string, filePath: string): Promise<void> {
    // Video processing is more time-consuming
    await this.fileQueue.add('process-video', {
      fileId,
      filePath,
      operations: ['transcode', 'generate-preview'],
    }, {
      attempts: 2, // Fewer retries for video (takes longer)
      timeout: 3600000, // 1 hour timeout
    });
  }
}
```

### File Processing Processor

```typescript
// src/files/file.processor.ts
import { Processor, Process } from '@nestjs/bull';
import { Job } from 'bull';
import { Injectable } from '@nestjs/common';
import * as sharp from 'sharp'; // Image processing library

@Processor('file-processing')
@Injectable()
export class FileProcessor {
  @Process('process-image')
  async handleImageProcessing(job: Job) {
    const { fileId, filePath, operations } = job.data;
    
    await job.progress(10);
    
    let image = sharp(filePath);
    
    // Resize if needed
    if (operations.includes('resize')) {
      await job.progress(30);
      image = image.resize(1920, 1080, { fit: 'inside' });
    }
    
    // Compress
    if (operations.includes('compress')) {
      await job.progress(50);
      await image.jpeg({ quality: 85 }).toFile(`processed/${fileId}.jpg`);
    }
    
    // Generate thumbnail
    if (operations.includes('generate-thumbnail')) {
      await job.progress(75);
      await sharp(filePath)
        .resize(300, 300, { fit: 'cover' })
        .toFile(`thumbnails/${fileId}.jpg`);
    }
    
    await job.progress(100);
    
    return {
      fileId,
      processed: true,
      paths: {
        original: filePath,
        processed: `processed/${fileId}.jpg`,
        thumbnail: `thumbnails/${fileId}.jpg`,
      },
    };
  }

  @Process('process-video')
  async handleVideoProcessing(job: Job) {
    const { fileId, filePath } = job.data;
    
    // Video processing logic (using ffmpeg or similar)
    // This is a long-running operation
    
    await job.progress(100);
    return { fileId, processed: true };
  }
}
```

---

## Best Practices

### 1. Keep Job Data Small

```typescript
// Good: Store reference, fetch full data in processor
await queue.add('process-user', { userId: 123 });

// Bad: Store entire user object
await queue.add('process-user', { 
  user: { /* large user object */ } 
});
```

### 2. Make Jobs Idempotent

Jobs should be safe to retry - processing twice should have the same effect as once.

```typescript
@Process('update-stats')
async handleUpdateStats(job: Job) {
  const { userId } = job.data;
  
  // Check if already processed (idempotency check)
  const existing = await this.statsRepository.findOne({ userId });
  if (existing && existing.lastUpdated > job.timestamp) {
    // Already processed by another job
    return existing;
  }
  
  // Process stats
  return await this.calculateStats(userId);
}
```

### 3. Use Appropriate Queue Names

```typescript
// Good: Descriptive queue names
'email-notifications'
'file-processing'
'data-sync'

// Bad: Vague names
'queue1'
'jobs'
'tasks'
```

### 4. Monitor Queue Health

```typescript
async getQueueStats() {
  const waiting = await this.emailQueue.getWaitingCount();
  const active = await this.emailQueue.getActiveCount();
  const completed = await this.emailQueue.getCompletedCount();
  const failed = await this.emailQueue.getFailedCount();
  
  return {
    waiting,
    active,
    completed,
    failed,
    total: waiting + active + completed + failed,
  };
}
```

### 5. Set Reasonable Timeouts

```typescript
await queue.add('long-operation', data, {
  timeout: 300000, // 5 minutes
});
```

### 6. Handle Queue Errors Gracefully

```typescript
@OnQueueError()
onError(error: Error) {
  // Don't let queue errors crash the application
  this.logger.error('Queue error:', error);
  // Send alert to monitoring service
}
```

---

## Mini Project: Email Notification System

Build a complete email notification system with queues.

### Requirements:
1. Send welcome emails on user registration
2. Send password reset emails (high priority)
3. Send weekly newsletter (scheduled)
4. Track email delivery status
5. Retry failed emails automatically
6. Monitor queue health

### Implementation Steps:

1. **Setup Bull Module** (as shown above)
2. **Create Email Service** with queue integration
3. **Create Email Processor** with real email sending
4. **Add Scheduled Newsletter** job
5. **Implement Monitoring** endpoint
6. **Add Error Handling** and retry logic

---

## Next Steps

✅ Understand queue concepts and Bull setup  
✅ Implement job producers and processors  
✅ Configure job scheduling with cron  
✅ Handle errors and retries  
✅ Move to Module 11: Comprehensive Testing  

---

*You now know how to process background jobs efficiently! 🚀*

