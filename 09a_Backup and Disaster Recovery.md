# Backup and Disaster Recovery Implementation

## Part 1: Database Backups

1. Database Backup Configuration:
   infrastructure/terraform/backup.tf:

   ```hcl
   # Backup vault for databases
   resource "aws_backup_vault" "database" {
     name = "gpx-tracker-backup-vault-${var.environment}"
     kms_key_arn = aws_kms_key.backup.arn

     tags = {
       Environment = var.environment
       Project     = "gpx-tracker"
     }
   }

   # KMS key for backup encryption
   resource "aws_kms_key" "backup" {
     description = "KMS key for backup encryption"
     deletion_window_in_days = 7
     enable_key_rotation = true

     tags = {
       Environment = var.environment
       Project     = "gpx-tracker"
     }
   }

   # Backup plan
   resource "aws_backup_plan" "database" {
     name = "gpx-tracker-backup-plan-${var.environment}"

     rule {
       rule_name = "daily_backup"
       target_vault_name = aws_backup_vault.database.name
       schedule = "cron(0 2 ? * * *)"

       lifecycle {
         delete_after = var.environment == "production" ? 30 : 7
       }

       copy_action {
         destination_vault_arn = var.environment == "production" ? aws_backup_vault.database_dr.arn : null
       }
     }

     # Weekly full backup for production
     dynamic "rule" {
       for_each = var.environment == "production" ? [1] : []
       content {
         rule_name = "weekly_backup"
         target_vault_name = aws_backup_vault.database.name
         schedule = "cron(0 2 ? * SAT *)"

         lifecycle {
           delete_after = 90
         }

         copy_action {
           destination_vault_arn = aws_backup_vault.database_dr.arn
         }
       }
     }

     tags = {
       Environment = var.environment
       Project     = "gpx-tracker"
     }
   }
   ```

2. Database Backup Scripts:
   backend/scripts/backup/database.py:

   ```python
   import subprocess
   import os
   from datetime import datetime
   import boto3
   import logging
   from botocore.exceptions import ClientError

   class DatabaseBackup:
       def __init__(self, db_url: str, bucket: str, prefix: str):
           self.db_url = db_url
           self.bucket = bucket
           self.prefix = prefix
           self.s3 = boto3.client('s3')

       def create_backup(self) -> str:
           """Create a database backup and upload to S3"""
           timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
           backup_file = f"backup_{timestamp}.sql.gz"

           try:
               # Create backup
               subprocess.run([
                   'pg_dump',
                   self.db_url,
                   '--clean',
                   '--no-owner',
                   '--no-privileges',
                   f'--file={backup_file}',
                   '--compress=9'
               ], check=True)

               # Upload to S3
               s3_key = f"{self.prefix}/{backup_file}"
               self.s3.upload_file(backup_file, self.bucket, s3_key)

               # Clean up local file
               os.remove(backup_file)

               logging.info(f"Backup created successfully: {s3_key}")
               return s3_key

           except subprocess.CalledProcessError as e:
               logging.error(f"Backup creation failed: {e}")
               raise
           except ClientError as e:
               logging.error(f"S3 upload failed: {e}")
               raise

       def restore_backup(self, backup_key: str) -> None:
           """Restore database from a backup"""
           try:
               # Download from S3
               local_file = os.path.basename(backup_key)
               self.s3.download_file(self.bucket, backup_key, local_file)

               # Restore backup
               subprocess.run([
                   'pg_restore',
                   '--clean',
                   '--no-owner',
                   '--no-privileges',
                   '--dbname', self.db_url,
                   local_file
               ], check=True)

               # Clean up
               os.remove(local_file)

               logging.info(f"Backup restored successfully: {backup_key}")

           except ClientError as e:
               logging.error(f"S3 download failed: {e}")
               raise
           except subprocess.CalledProcessError as e:
               logging.error(f"Restore failed: {e}")
               raise

       def list_backups(self) -> list:
           """List available backups"""
           try:
               response = self.s3.list_objects_v2(
                   Bucket=self.bucket,
                   Prefix=self.prefix
               )

               backups = []
               for obj in response.get('Contents', []):
                   backups.append({
                       'key': obj['Key'],
                       'size': obj['Size'],
                       'last_modified': obj['LastModified']
                   })

               return sorted(backups, key=lambda x: x['last_modified'], reverse=True)

           except ClientError as e:
               logging.error(f"Failed to list backups: {e}")
               raise

       def validate_backup(self, backup_key: str) -> bool:
           """Validate backup integrity"""
           try:
               # Download backup
               local_file = os.path.basename(backup_key)
               self.s3.download_file(self.bucket, backup_key, local_file)

               # Verify backup
               result = subprocess.run([
                   'pg_restore',
                   '--list',
                   local_file
               ], capture_output=True)

               # Clean up
               os.remove(local_file)

               return result.returncode == 0

           except Exception as e:
               logging.error(f"Backup validation failed: {e}")
               return False
   ```

## Part 2: File System and State Backups",

1. File System Backup Service:
   backend/app/services/backup/file_system.py:

   ```python
   from typing import List, Dict, Any
   import boto3
   import os
   from datetime import datetime, timedelta
   import hashlib
   from concurrent.futures import ThreadPoolExecutor
   import logging
   from app.core.config import settings

   class FileSystemBackup:
       def __init__(self, source_bucket: str, backup_bucket: str, max_workers: int = 4):
           self.s3 = boto3.client('s3')
           self.source_bucket = source_bucket
           self.backup_bucket = backup_bucket
           self.max_workers = max_workers

       def _calculate_checksum(self, bucket: str, key: str) -> str:
           """Calculate MD5 checksum of S3 object"""
           try:
               response = self.s3.get_object(Bucket=bucket, Key=key)
               return response['ETag'].strip('"')
           except Exception as e:
               logging.error(f"Error calculating checksum for {key}: {e}")
               raise

       def _backup_file(self, file_info: Dict[str, Any]) -> Dict[str, Any]:
           """Backup single file with verification"""
           try:
               source_key = file_info['Key']
               backup_key = f"backup_{datetime.now().strftime('%Y%m%d')}/{source_key}"

               # Copy file to backup bucket
               self.s3.copy_object(
                   CopySource={'Bucket': self.source_bucket, 'Key': source_key},
                   Bucket=self.backup_bucket,
                   Key=backup_key,
                   MetadataDirective='COPY',
                   TaggingDirective='COPY'
               )

               # Verify checksums
               source_checksum = self._calculate_checksum(self.source_bucket, source_key)
               backup_checksum = self._calculate_checksum(self.backup_bucket, backup_key)

               if source_checksum != backup_checksum:
                   raise ValueError(f"Checksum mismatch for {source_key}")

               return {
                   'source_key': source_key,
                   'backup_key': backup_key,
                   'size': file_info['Size'],
                   'status': 'success',
                   'checksum': source_checksum
               }

           except Exception as e:
               logging.error(f"Backup failed for {source_key}: {e}")
               return {
                   'source_key': source_key,
                   'status': 'failed',
                   'error': str(e)
               }

       def create_backup(self, prefix: str = None) -> Dict[str, Any]:
           """Create backup of all files or files under specific prefix"""
           try:
               # List all files in source bucket
               paginator = self.s3.get_paginator('list_objects_v2')
               files_to_backup = []

               for page in paginator.paginate(Bucket=self.source_bucket, Prefix=prefix or ''):
                   files_to_backup.extend(page.get('Contents', []))

               # Backup files in parallel
               results = []
               with ThreadPoolExecutor(max_workers=self.max_workers) as executor:
                   futures = [executor.submit(self._backup_file, file_info)
                             for file_info in files_to_backup]

                   for future in futures:
                       results.append(future.result())

               # Generate backup report
               successful = [r for r in results if r['status'] == 'success']
               failed = [r for r in results if r['status'] == 'failed']

               return {
                   'timestamp': datetime.now().isoformat(),
                   'total_files': len(files_to_backup),
                   'successful': len(successful),
                   'failed': len(failed),
                   'total_size': sum(r['size'] for r in successful),
                   'failed_files': [r['source_key'] for r in failed],
                   'backup_prefix': f"backup_{datetime.now().strftime('%Y%m%d')}"
               }

           except Exception as e:
               logging.error(f"Backup process failed: {e}")
               raise

       def restore_files(self, backup_prefix: str, target_prefix: str = None) -> Dict[str, Any]:
           """Restore files from backup"""
           try:
               # List all files in backup
               paginator = self.s3.get_paginator('list_objects_v2')
               files_to_restore = []

               for page in paginator.paginate(Bucket=self.backup_bucket, Prefix=backup_prefix):
                   files_to_restore.extend(page.get('Contents', []))

               restored_files = []
               failed_files = []

               for file_info in files_to_restore:
                   try:
                       source_key = file_info['Key']
                       target_key = source_key.replace(backup_prefix + '/', target_prefix or '')

                       # Copy file back to source bucket
                       self.s3.copy_object(
                           CopySource={'Bucket': self.backup_bucket, 'Key': source_key},
                           Bucket=self.source_bucket,
                           Key=target_key
                       )

                       restored_files.append({
                           'source_key': source_key,
                           'target_key': target_key,
                           'size': file_info['Size']
                       })

                   except Exception as e:
                       logging.error(f"Failed to restore {source_key}: {e}")
                       failed_files.append({
                           'key': source_key,
                           'error': str(e)
                       })

               return {
                   'timestamp': datetime.now().isoformat(),
                   'backup_prefix': backup_prefix,
                   'target_prefix': target_prefix,
                   'total_files': len(files_to_restore),
                   'restored': len(restored_files),
                   'failed': len(failed_files),
                   'total_size_restored': sum(f['size'] for f in restored_files),
                   'failed_files': failed_files
               }

           except Exception as e:
               logging.error(f"Restore process failed: {e}")
               raise
   ```

2. Backup Management CLI:
   backend/scripts/backup/manager.py:

   ```python
   import click
   import json
   from datetime import datetime
   from app.services.backup.database import DatabaseBackup
   from app.services.backup.file_system import FileSystemBackup
   from app.core.config import settings

   @click.group()
   def cli():
       """GPX Tracker Backup Management CLI"""
       pass

   @cli.command()
   @click.option('--type', type=click.Choice(['full', 'db', 'files']), default='full')
   @click.option('--prefix', help='Backup specific prefix for files')
   def create(type, prefix):
       """Create a new backup"""
       results = {}

       if type in ['full', 'db']:
           db_backup = DatabaseBackup(
               settings.DATABASE_URL,
               settings.BACKUP_BUCKET,
               'database'
           )
           results['database'] = db_backup.create_backup()

       if type in ['full', 'files']:
           fs_backup = FileSystemBackup(
               settings.STORAGE_BUCKET,
               settings.BACKUP_BUCKET
           )
           results['files'] = fs_backup.create_backup(prefix)

       click.echo(json.dumps(results, indent=2))

   @cli.command()
   @click.option('--backup-key', required=True, help='Backup key to restore from')
   @click.option('--type', type=click.Choice(['full', 'db', 'files']), default='full')
   @click.option('--target-prefix', help='Target prefix for file restoration')
   def restore(backup_key, type, target_prefix):
       """Restore from backup"""
       if not click.confirm('This will overwrite existing data. Continue?'):
           return

       results = {}

       if type in ['full', 'db']:
           db_backup = DatabaseBackup(
               settings.DATABASE_URL,
               settings.BACKUP_BUCKET,
               'database'
           )
           results['database'] = db_backup.restore_backup(backup_key)

       if type in ['full', 'files']:
           fs_backup = FileSystemBackup(
               settings.STORAGE_BUCKET,
               settings.BACKUP_BUCKET
           )
           results['files'] = fs_backup.restore_files(backup_key, target_prefix)

       click.echo(json.dumps(results, indent=2))

   @cli.command()
   @click.option('--days', default=7, help='List backups from the last N days')
   def list(days):
       """List available backups"""
       db_backup = DatabaseBackup(
           settings.DATABASE_URL,
           settings.BACKUP_BUCKET,
           'database'
       )
       backups = db_backup.list_backups()

       click.echo("Available backups:")
       for backup in backups:
           click.echo(f"- {backup['key']} ({backup['size']} bytes, {backup['last_modified']})")

   if __name__ == '__main__':
       cli()
   ```

## Part 3: Recovery Automation",

1. Recovery Automation Service:
   backend/app/services/backup/recovery.py:

   ```python
   from typing import Dict, Any, List
   import boto3
   import logging
   from datetime import datetime
   from app.services.backup.database import DatabaseBackup
   from app.services.backup.file_system import FileSystemBackup
   from app.core.monitoring import Monitoring

   class RecoveryAutomation:
       def __init__(self):
           self.sns = boto3.client('sns')
           self.monitoring = Monitoring()
           self.db_backup = DatabaseBackup(
               settings.DATABASE_URL,
               settings.BACKUP_BUCKET,
               'database'
           )
           self.fs_backup = FileSystemBackup(
               settings.STORAGE_BUCKET,
               settings.BACKUP_BUCKET
           )

       async def execute_recovery_plan(self, incident_type: str) -> Dict[str, Any]:
           """Execute automated recovery based on incident type"""
           start_time = datetime.now()
           recovery_id = f"recovery_{start_time.strftime('%Y%m%d_%H%M%S')}"

           try:
               logging.info(f"Starting recovery process: {recovery_id}")
               self.monitoring.track_event('recovery_started', {'incident_type': incident_type})

               # Get latest valid backup
               latest_backup = self._get_latest_valid_backup()
               if not latest_backup:
                   raise ValueError("No valid backup found")

               # Execute recovery steps
               recovery_steps = self._get_recovery_steps(incident_type)
               results = await self._execute_steps(recovery_steps, latest_backup)

               # Validate recovery
               validation_results = await self._validate_recovery(results)

               end_time = datetime.now()
               duration = (end_time - start_time).total_seconds()

               recovery_report = {
                   'recovery_id': recovery_id,
                   'incident_type': incident_type,
                   'start_time': start_time.isoformat(),
                   'end_time': end_time.isoformat(),
                   'duration': duration,
                   'status': 'success' if validation_results['success'] else 'failed',
                   'steps': results,
                   'validation': validation_results
               }

               # Send notification
               self._send_recovery_notification(recovery_report)
               self.monitoring.track_event('recovery_completed', recovery_report)

               return recovery_report

           except Exception as e:
               error_report = {
                   'recovery_id': recovery_id,
                   'incident_type': incident_type,
                   'error': str(e),
                   'status': 'failed'
               }
               self._send_recovery_notification(error_report)
               self.monitoring.track_event('recovery_failed', error_report)
               raise

       def _get_recovery_steps(self, incident_type: str) -> List[Dict[str, Any]]:
           """Get recovery steps based on incident type"""
           recovery_plans = {
               'database_corruption': [
                   {
                       'step': 'stop_application',
                       'service': 'application',
                       'action': 'stop'
                   },
                   {
                       'step': 'restore_database',
                       'service': 'database',
                       'action': 'restore'
                   },
                   {
                       'step': 'validate_database',
                       'service': 'database',
                       'action': 'validate'
                   },
                   {
                       'step': 'start_application',
                       'service': 'application',
                       'action': 'start'
                   }
               ],
               'storage_failure': [
                   {
                       'step': 'stop_uploads',
                       'service': 'storage',
                       'action': 'disable_uploads'
                   },
                   {
                       'step': 'restore_files',
                       'service': 'storage',
                       'action': 'restore'
                   },
                   {
                       'step': 'validate_files',
                       'service': 'storage',
                       'action': 'validate'
                   },
                   {
                       'step': 'enable_uploads',
                       'service': 'storage',
                       'action': 'enable_uploads'
                   }
               ],
               'full_system': [
                   {
                       'step': 'stop_all_services',
                       'service': 'system',
                       'action': 'stop_all'
                   },
                   {
                       'step': 'restore_database',
                       'service': 'database',
                       'action': 'restore'
                   },
                   {
                       'step': 'restore_files',
                       'service': 'storage',
                       'action': 'restore'
                   },
                   {
                       'step': 'validate_all',
                       'service': 'system',
                       'action': 'validate_all'
                   },
                   {
                       'step': 'start_all_services',
                       'service': 'system',
                       'action': 'start_all'
                   }
               ]
           }

           return recovery_plans.get(incident_type, [])

       async def _execute_steps(self, steps: List[Dict[str, Any]], backup_info: Dict[str, Any])
           -> List[Dict[str, Any]]:
           """Execute recovery steps in sequence"""
           results = []

           for step in steps:
               try:
                   start_time = datetime.now()
                   logging.info(f"Executing step: {step['step']}")

                   if step['service'] == 'database':
                       if step['action'] == 'restore':
                           await self.db_backup.restore_backup(backup_info['database_key'])
                       elif step['action'] == 'validate':
                           await self.db_backup.validate_backup(backup_info['database_key'])

                   elif step['service'] == 'storage':
                       if step['action'] == 'restore':
                           await self.fs_backup.restore_files(
                               backup_info['files_prefix'],
                               target_prefix=None
                           )

                   elif step['service'] == 'system':
                       if step['action'] == 'stop_all':
                           await self._stop_all_services()
                       elif step['action'] == 'start_all':
                           await self._start_all_services()

                   end_time = datetime.now()
                   duration = (end_time - start_time).total_seconds()

                   results.append({
                       'step': step['step'],
                       'status': 'success',
                       'duration': duration
                   })

               except Exception as e:
                   results.append({
                       'step': step['step'],
                       'status': 'failed',
                       'error': str(e)
                   })
                   raise

           return results
   ```

## Part 4: Automated Solutions and Testing

1. Automated Backup Solution:
   backend/app/services/backup/automation.py:

   ```python
   import asyncio
   from datetime import datetime, timedelta
   import logging
   from typing import List, Dict, Any
   from croniter import croniter
   from app.core.config import settings
   from app.services.monitoring import Monitoring

   class BackupAutomation:
       def __init__(self):
           self.monitor = Monitoring()
           self.schedules = {
               'daily_full': {
                   'cron': '0 1 * * *',  # 1 AM daily
                   'type': 'full',
                   'retention_days': 7
               },
               'hourly_incremental': {
                   'cron': '0 * * * *',  # Every hour
                   'type': 'incremental',
                   'retention_days': 2
               },
               'weekly_archive': {
                   'cron': '0 2 * * 0',  # 2 AM on Sundays
                   'type': 'archive',
                   'retention_days': 90
               }
           }

       async def run_scheduled_backups(self):
           """Run continuous backup scheduling"""
           while True:
               try:
                   await self._check_and_execute_backups()
                   await asyncio.sleep(60)  # Check every minute
               except Exception as e:
                   logging.error(f"Backup scheduling error: {e}")
                   self.monitor.alert('backup_scheduler_error', str(e))

       async def _check_and_execute_backups(self):
           now = datetime.now()
           for schedule_name, config in self.schedules.items():
               cron = croniter(config['cron'], now)
               next_run = cron.get_prev(datetime)

               if (now - next_run) < timedelta(minutes=1):
                   try:
                       await self._execute_backup(schedule_name, config)
                   except Exception as e:
                       logging.error(f"Backup execution error: {schedule_name} - {e}")
                       self.monitor.alert('backup_execution_error', {
                           'schedule': schedule_name,
                           'error': str(e)
                       })

       async def _execute_backup(self, schedule_name: str, config: Dict[str, Any]):
           start_time = datetime.now()
           backup_id = f"{schedule_name}_{start_time.strftime('%Y%m%d_%H%M%S')}"

           try:
               self.monitor.event('backup_started', {
                   'backup_id': backup_id,
                   'type': config['type'],
                   'schedule': schedule_name
               })

               # Execute backup based on type
               if config['type'] == 'full':
                   result = await self._run_full_backup(backup_id)
               elif config['type'] == 'incremental':
                   result = await self._run_incremental_backup(backup_id)
               else:  # archive
                   result = await self._run_archive_backup(backup_id)

               # Clean up old backups
               await self._cleanup_old_backups(config['type'],
                                             config['retention_days'])

               duration = (datetime.now() - start_time).total_seconds()
               result.update({
                   'duration': duration,
                   'schedule': schedule_name
               })

               self.monitor.event('backup_completed', result)
               return result

           except Exception as e:
               self.monitor.event('backup_failed', {
                   'backup_id': backup_id,
                   'error': str(e),
                   'schedule': schedule_name
               })
               raise

       async def verify_backup_integrity(self, backup_id: str) -> bool:
           """Verify the integrity of a backup"""
           try:
               # Perform various integrity checks
               checks = await asyncio.gather(
                   self._verify_database_backup(backup_id),
                   self._verify_file_checksums(backup_id),
                   self._verify_backup_metadata(backup_id)
               )

               return all(checks)

           except Exception as e:
               logging.error(f"Backup verification failed: {backup_id} - {e}")
               return False
   ```

2. Recovery Testing Protocol Implementation:
   backend/app/services/backup/testing.py:

   ```python
   from typing import List, Dict, Any
   from datetime import datetime
   import logging
   import json
   from app.core.config import settings

   class RecoveryTesting:
       def __init__(self):
           self.test_scenarios = self._load_test_scenarios()
           self.results_history: List[Dict[str, Any]] = []

       def _load_test_scenarios(self) -> Dict[str, Any]:
           """Load test scenarios from configuration"""
           with open('config/recovery_tests.json', 'r') as f:
               return json.load(f)

       async def run_test_scenario(self, scenario_name: str) -> Dict[str, Any]:
           """Run a specific recovery test scenario"""
           scenario = self.test_scenarios.get(scenario_name)
           if not scenario:
               raise ValueError(f"Unknown test scenario: {scenario_name}")

           test_id = f"test_{scenario_name}_{datetime.now().strftime('%Y%m%d_%H%M%S')}"
           start_time = datetime.now()

           try:
               # Create isolated test environment
               test_env = await self._create_test_environment()

               # Execute test steps
               results = []
               for step in scenario['steps']:
                   step_result = await self._execute_test_step(step, test_env)
                   results.append(step_result)
                   if not step_result['success'] and step.get('critical', False):
                       break

               # Cleanup test environment
               await self._cleanup_test_environment(test_env)

               # Calculate success metrics
               success_rate = len([r for r in results if r['success']]) / len(results)
               duration = (datetime.now() - start_time).total_seconds()

               test_result = {
                   'test_id': test_id,
                   'scenario': scenario_name,
                   'success_rate': success_rate,
                   'duration': duration,
                   'steps': results,
                   'timestamp': start_time.isoformat()
               }

               self.results_history.append(test_result)
               return test_result

           except Exception as e:
               logging.error(f"Test scenario failed: {scenario_name} - {e}")
               raise

       async def validate_recovery_procedures(self) -> Dict[str, Any]:
           """Run comprehensive recovery procedure validation"""
           validation_results = {
               'timestamp': datetime.now().isoformat(),
               'scenarios_tested': 0,
               'successful_scenarios': 0,
               'failed_scenarios': 0,
               'results': []
           }

           for scenario_name in self.test_scenarios.keys():
               try:
                   result = await self.run_test_scenario(scenario_name)
                   validation_results['scenarios_tested'] += 1

                   if result['success_rate'] >= 0.95:  # 95% success threshold
                       validation_results['successful_scenarios'] += 1
                   else:
                       validation_results['failed_scenarios'] += 1

                   validation_results['results'].append(result)

               except Exception as e:
                   validation_results['failed_scenarios'] += 1
                   validation_results['results'].append({
                       'scenario': scenario_name,
                       'error': str(e),
                       'success_rate': 0
                   })

           return validation_results

       async def generate_recovery_test_report(self) -> Dict[str, Any]:
           """Generate comprehensive recovery test report"""
           validation = await self.validate_recovery_procedures()

           return {
               'summary': {
                   'total_scenarios': validation['scenarios_tested'],
                   'successful_scenarios': validation['successful_scenarios'],
                   'success_rate': validation['successful_scenarios'] /
                                  validation['scenarios_tested'] if validation['scenarios_tested'] > 0 else 0,
                   'timestamp': datetime.now().isoformat()
               },
               'detailed_results': validation['results'],
               'recommendations': self._generate_recommendations(validation)
           }
   ```
