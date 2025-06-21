import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from email.mime.application import MIMEApplication
import getpass
from rich.console import Console
from rich.panel import Panel
from rich.prompt import Prompt, Confirm
from rich.progress import Progress
from rich.table import Table
from rich.text import Text
import os
from typing import List

# Initialize rich console
console = Console()

class EmailSender:
    def __init__(self):
        self.smtp_config = {
            'Gmail': {'server': 'smtp.gmail.com', 'port': 587},
            'Outlook': {'server': 'smtp-mail.outlook.com', 'port': 587},
            'Yahoo': {'server': 'smtp.mail.yahoo.com', 'port': 587},
            'Custom': {'server': '', 'port': ''}
        }
        
    def display_banner(self):
        banner = Text("""
  ______ _____ _      _____ _____ _____ _____ 
 |  ____/ ____| |    |_   _|  __ \_   _/ ____|
 | |__ | |    | |      | | | |__) || || |     
 |  __|| |    | |      | | |  ___/ | || |     
 | |___| |____| |____ _| |_| |    _| || |____ 
 |______\_____|______|_____|_|   |_____\_____|
        """, style="bold cyan")
        console.print(banner)
        console.print(Panel.fit("Secure Email Sending Tool", style="bold blue"))

    def get_email_details(self) -> dict:
        """Collect email details from user with rich interface"""
        console.print(Panel("📧 Email Details", style="bold green"))
        
        # Select email provider
        provider = Prompt.ask(
            "Select your email provider",
            choices=list(self.smtp_config.keys()),
            default="Gmail"
        )
        
        if provider == "Custom":
            self.smtp_config[provider]['server'] = Prompt.ask("Enter SMTP server address")
            self.smtp_config[provider]['port'] = int(Prompt.ask("Enter SMTP port"))
        
        details = {
            'sender': Prompt.ask("📤 Your email address"),
            'recipients': [r.strip() for r in Prompt.ask("📨 Recipients (comma separated)").split(',')],
            'subject': Prompt.ask("📝 Subject"),
            'message': self.get_message_content(),
            'attachments': self.get_attachments(),
            'server': self.smtp_config[provider]['server'],
            'port': self.smtp_config[provider]['port'],
            'use_tls': True
        }
        
        return details
    
    def get_message_content(self) -> str:
        """Get message content with option for HTML or plain text"""
        message_type = Prompt.ask(
            "Select message type",
            choices=["plain", "html"],
            default="plain"
        )
        
        console.print("\n💬 Enter your message (press Enter then Ctrl+D when done):", style="bold")
        lines = []
        while True:
            try:
                line = console.input()
                lines.append(line)
            except EOFError:
                break
        
        message = '\n'.join(lines)
        
        if message_type == "html":
            # Basic HTML formatting if needed
            message = f"<html><body><p>{message.replace('\n', '<br>')}</p></body></html>"
        
        return message
    
    def get_attachments(self) -> List[str]:
        """Get attachment files with validation"""
        attachments = []
        while True:
            attach = Confirm.ask("📎 Add an attachment?")
            if not attach:
                break
            file_path = Prompt.ask("Enter file path")
            if os.path.exists(file_path):
                attachments.append(file_path)
                console.print(f"✓ Added: {file_path}", style="green")
            else:
                console.print(f"⚠ File not found: {file_path}", style="bold yellow")
        
        return attachments if attachments else None
    
    def send_email(self, email_details: dict):
        """Send the email with progress indication"""
        msg = MIMEMultipart()
        msg['From'] = email_details['sender']
        msg['To'] = ', '.join(email_details['recipients'])
        msg['Subject'] = email_details['subject']
        
        # Attach message
        content_type = 'html' if email_details['message'].startswith('<html>') else 'plain'
        msg.attach(MIMEText(email_details['message'], content_type))
        
        # Attach files
        if email_details['attachments']:
            with Progress() as progress:
                task = progress.add_task("[cyan]Attaching files...", total=len(email_details['attachments']))
                for file_path in email_details['attachments']:
                    try:
                        with open(file_path, 'rb') as file:
                            part = MIMEApplication(file.read(), Name=os.path.basename(file_path))
                        part['Content-Disposition'] = f'attachment; filename="{os.path.basename(file_path)}"'
                        msg.attach(part)
                        progress.update(task, advance=1, description=f"[green]Attached {os.path.basename(file_path)}")
                    except Exception as e:
                        progress.update(task, description=f"[red]Failed to attach {file_path}")
                        console.print(f"Error attaching {file_path}: {str(e)}", style="bold red")
        
        # Get password securely
        password = getpass.getpass(f"🔑 Enter password for {email_details['sender']}: ")
        
        # Send email with progress indication
        try:
            with Progress() as progress:
                task = progress.add_task("[blue]Sending email...", total=3)
                
                progress.update(task, advance=1, description="[cyan]Connecting to server...")
                with smtplib.SMTP(email_details['server'], email_details['port']) as server:
                    if email_details['use_tls']:
                        server.starttls()
                    
                    progress.update(task, advance=1, description="[cyan]Authenticating...")
                    server.login(email_details['sender'], password)
                    
                    progress.update(task, advance=1, description="[green]Sending message...")
                    server.sendmail(email_details['sender'], email_details['recipients'], msg.as_string())
            
            console.print(Panel.fit("✅ Email sent successfully!", style="bold green"))
            
            # Display summary table
            summary = Table(title="Email Summary", show_header=True, header_style="bold magenta")
            summary.add_column("Field", style="cyan")
            summary.add_column("Value", style="white")
            
            summary.add_row("From", email_details['sender'])
            summary.add_row("To", '\n'.join(email_details['recipients']))
            summary.add_row("Subject", email_details['subject'])
            summary.add_row("Attachments", str(len(email_details['attachments'])) if email_details['attachments'] else "None")
            
            console.print(summary)
            
        except smtplib.SMTPAuthenticationError:
            console.print(Panel.fit("❌ Authentication failed. Check your email and password.", style="bold red"))
        except Exception as e:
            console.print(Panel.fit(f"❌ Error sending email: {str(e)}", style="bold red"))

def main():
    sender = EmailSender()
    sender.display_banner()
    
    try:
        while True:
            email_details = sender.get_email_details()
            sender.send_email(email_details)
            
            if not Confirm.ask("Send another email?"):
                break
                
    except KeyboardInterrupt:
        console.print("\n👋 Goodbye!", style="bold yellow")
    except Exception as e:
        console.print(f"An unexpected error occurred: {str(e)}", style="bold red")

if __name__ == "__main__":
    main(
