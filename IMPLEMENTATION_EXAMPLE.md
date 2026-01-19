# Implementation Example: Config App Structure

This document shows the actual code structure for implementing the configuration system.

## Step 1: Create the Config App

```bash
python manage.py startapp config
```

## Step 2: Config Models (config/models.py)

```python
from django.db import models
from django.core.cache import cache
from django.core.validators import MinValueValidator, MaxValueValidator

class SiteConfig(models.Model):
    """Main site configuration"""
    site_name = models.CharField(max_length=100, default="FastCart")
    site_tagline = models.CharField(max_length=200, blank=True)
    logo = models.ImageField(upload_to='config/', blank=True, null=True)
    favicon = models.ImageField(upload_to='config/', blank=True, null=True)
    primary_color = models.CharField(max_length=7, default="#007bff")
    secondary_color = models.CharField(max_length=7, default="#6c757d")
    contact_email = models.EmailField(default="support@example.com")
    contact_phone = models.CharField(max_length=20, blank=True)
    address = models.TextField(blank=True)
    
    class Meta:
        verbose_name_plural = "Site Configuration"
    
    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        # Clear cache when config changes
        cache.delete('site_config')
    
    @classmethod
    def get_config(cls):
        """Get cached site config"""
        config = cache.get('site_config')
        if config is None:
            config = cls.objects.first()
            if config:
                cache.set('site_config', config, 3600)  # Cache for 1 hour
        return config

class FeatureFlag(models.Model):
    """Feature flags for enabling/disabling features"""
    name = models.CharField(max_length=100, unique=True)
    enabled = models.BooleanField(default=True)
    description = models.TextField(blank=True)
    
    class Meta:
        verbose_name_plural = "Feature Flags"
    
    def __str__(self):
        return f"{self.name} ({'Enabled' if self.enabled else 'Disabled'})"
    
    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        cache.delete(f'feature_flag_{self.name}')

class ServiceFeeConfig(models.Model):
    """Service fee configuration"""
    fee_percentage = models.DecimalField(
        max_digits=5, 
        decimal_places=2, 
        default=5.00,
        validators=[MinValueValidator(0), MaxValueValidator(100)]
    )
    min_fee = models.DecimalField(max_digits=10, decimal_places=2, default=0.00)
    max_fee = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    active = models.BooleanField(default=True)
    
    class Meta:
        verbose_name_plural = "Service Fee Configuration"
    
    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        cache.delete('service_fee_config')
    
    @classmethod
    def get_config(cls):
        config = cache.get('service_fee_config')
        if config is None:
            config = cls.objects.filter(active=True).first()
            if config:
                cache.set('service_fee_config', config, 3600)
        return config

class TaxConfig(models.Model):
    """Tax rates by country"""
    country = models.CharField(max_length=100, unique=True)
    tax_rate = models.DecimalField(
        max_digits=5, 
        decimal_places=2,
        validators=[MinValueValidator(0), MaxValueValidator(100)]
    )
    active = models.BooleanField(default=True)
    
    class Meta:
        verbose_name_plural = "Tax Configuration"
        ordering = ['country']
    
    def __str__(self):
        return f"{self.country} - {self.tax_rate}%"

class ShippingService(models.Model):
    """Shipping service providers"""
    name = models.CharField(max_length=100, unique=True)
    code = models.CharField(max_length=50, unique=True)
    active = models.BooleanField(default=True)
    api_key = models.CharField(max_length=255, blank=True, null=True)
    tracking_url_template = models.URLField(blank=True, null=True)
    priority = models.IntegerField(default=0)  # Display order
    
    class Meta:
        verbose_name_plural = "Shipping Services"
        ordering = ['priority', 'name']
    
    def __str__(self):
        return self.name

class PaymentGateway(models.Model):
    """Payment gateway configuration"""
    PAYMENT_METHODS = (
        ("PayPal", "PayPal"),
        ("Stripe", "Stripe"),
        ("Flutterwave", "Flutterwave"),
        ("Paystack", "Paystack"),
        ("RazorPay", "RazorPay"),
    )
    
    name = models.CharField(max_length=50, choices=PAYMENT_METHODS, unique=True)
    enabled = models.BooleanField(default=False)
    public_key = models.CharField(max_length=255, blank=True)
    secret_key = models.CharField(max_length=255, blank=True)
    test_mode = models.BooleanField(default=True)
    priority = models.IntegerField(default=0)  # Display order
    
    class Meta:
        verbose_name_plural = "Payment Gateways"
        ordering = ['priority', 'name']
    
    def __str__(self):
        return f"{self.name} ({'Enabled' if self.enabled else 'Disabled'})"

class CurrencyConfig(models.Model):
    """Currency configuration"""
    code = models.CharField(max_length=3, unique=True)  # USD, EUR, NGN, etc.
    symbol = models.CharField(max_length=5)
    name = models.CharField(max_length=50)
    is_default = models.BooleanField(default=False)
    exchange_rate = models.DecimalField(max_digits=10, decimal_places=4, default=1.0000)
    active = models.BooleanField(default=True)
    
    class Meta:
        verbose_name_plural = "Currencies"
        ordering = ['-is_default', 'code']
    
    def __str__(self):
        return f"{self.code} - {self.name}"
    
    def save(self, *args, **kwargs):
        # Ensure only one default currency
        if self.is_default:
            CurrencyConfig.objects.filter(is_default=True).update(is_default=False)
        super().save(*args, **kwargs)
```

## Step 3: Config Admin (config/admin.py)

```python
from django.contrib import admin
from .models import (
    SiteConfig, FeatureFlag, ServiceFeeConfig, 
    TaxConfig, ShippingService, PaymentGateway, CurrencyConfig
)

@admin.register(SiteConfig)
class SiteConfigAdmin(admin.ModelAdmin):
    list_display = ['site_name', 'contact_email', 'contact_phone']
    fieldsets = (
        ('Basic Information', {
            'fields': ('site_name', 'site_tagline')
        }),
        ('Branding', {
            'fields': ('logo', 'favicon', 'primary_color', 'secondary_color')
        }),
        ('Contact Information', {
            'fields': ('contact_email', 'contact_phone', 'address')
        }),
    )
    
    def has_add_permission(self, request):
        # Only allow one site config
        return not SiteConfig.objects.exists()
    
    def has_delete_permission(self, request, obj=None):
        return False

@admin.register(FeatureFlag)
class FeatureFlagAdmin(admin.ModelAdmin):
    list_display = ['name', 'enabled', 'description']
    list_editable = ['enabled']
    list_filter = ['enabled']
    search_fields = ['name', 'description']

@admin.register(ServiceFeeConfig)
class ServiceFeeConfigAdmin(admin.ModelAdmin):
    list_display = ['fee_percentage', 'min_fee', 'max_fee', 'active']
    list_editable = ['active']
    
    def has_add_permission(self, request):
        return not ServiceFeeConfig.objects.exists()

@admin.register(TaxConfig)
class TaxConfigAdmin(admin.ModelAdmin):
    list_display = ['country', 'tax_rate', 'active']
    list_editable = ['active']
    list_filter = ['active']
    search_fields = ['country']

@admin.register(ShippingService)
class ShippingServiceAdmin(admin.ModelAdmin):
    list_display = ['name', 'code', 'active', 'priority']
    list_editable = ['active', 'priority']
    list_filter = ['active']

@admin.register(PaymentGateway)
class PaymentGatewayAdmin(admin.ModelAdmin):
    list_display = ['name', 'enabled', 'test_mode', 'priority']
    list_editable = ['enabled', 'test_mode', 'priority']
    list_filter = ['enabled', 'test_mode']
    fieldsets = (
        ('Basic', {
            'fields': ('name', 'enabled', 'priority')
        }),
        ('API Keys', {
            'fields': ('public_key', 'secret_key', 'test_mode'),
            'classes': ('collapse',)
        }),
    )

@admin.register(CurrencyConfig)
class CurrencyConfigAdmin(admin.ModelAdmin):
    list_display = ['code', 'name', 'symbol', 'is_default', 'exchange_rate', 'active']
    list_editable = ['is_default', 'active', 'exchange_rate']
    list_filter = ['active', 'is_default']
```

## Step 4: Settings Manager (config/settings_manager.py)

```python
from django.core.cache import cache
from .models import FeatureFlag, SiteConfig, ServiceFeeConfig

def is_feature_enabled(feature_name):
    """Check if a feature is enabled"""
    cache_key = f'feature_flag_{feature_name}'
    enabled = cache.get(cache_key)
    
    if enabled is None:
        try:
            flag = FeatureFlag.objects.get(name=feature_name)
            enabled = flag.enabled
            cache.set(cache_key, enabled, 3600)
        except FeatureFlag.DoesNotExist:
            # Default to False if flag doesn't exist
            enabled = False
            cache.set(cache_key, enabled, 3600)
    
    return enabled

def get_site_config():
    """Get site configuration"""
    return SiteConfig.get_config()

def get_service_fee_config():
    """Get service fee configuration"""
    return ServiceFeeConfig.get_config()

def calculate_service_fee(order_total):
    """Calculate service fee based on config"""
    config = get_service_fee_config()
    if not config or not config.active:
        return 0
    
    fee = (order_total * config.fee_percentage) / 100
    
    if config.min_fee and fee < config.min_fee:
        fee = config.min_fee
    
    if config.max_fee and fee > config.max_fee:
        fee = config.max_fee
    
    return fee
```

## Step 5: Updated Plugin Files

### plugin/service_fee.py (Updated)
```python
from decimal import Decimal
from config.settings_manager import calculate_service_fee as config_calculate_service_fee

def calculate_service_fee(order_total):
    """Calculate service fee - uses config if available, falls back to default"""
    try:
        return config_calculate_service_fee(order_total)
    except:
        # Fallback to default 5% if config not available
        return Decimal(order_total) * Decimal(5) / 100
```

### plugin/tax_calculation.py (Updated)
```python
from config.models import TaxConfig

def tax_calculation(country, order_total):
    """Calculate tax based on country - uses config if available"""
    try:
        tax_config = TaxConfig.objects.filter(country=country, active=True).first()
        if tax_config:
            return float(order_total) * float(tax_config.tax_rate) / 100
    except:
        pass
    
    # Fallback to hardcoded rates if config not available
    from plugin.countries import countries
    for c in countries():
        if country == c['country']:
            return int(float(c['tax_rate'])) / 100 * float(order_total)
    
    return 0
```

## Step 6: Updated Settings Structure

### ecom_prj/settings/__init__.py
```python
import os
from .base import *

# Determine which settings to use
ENVIRONMENT = os.environ.get('DJANGO_ENVIRONMENT', 'development')

if ENVIRONMENT == 'production':
    from .production import *
elif ENVIRONMENT == 'staging':
    from .staging import *
else:
    from .development import *
```

### ecom_prj/settings/base.py
```python
from pathlib import Path
from environs import Env

env = Env()
env.read_env()

BASE_DIR = Path(__file__).resolve().parent.parent.parent

# Core settings
SECRET_KEY = env('SECRET_KEY', 'your-secret-key-here')
DEBUG = False  # Override in dev/staging
ALLOWED_HOSTS = []

# Apps
INSTALLED_APPS = [
    'jazzmin',
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.humanize',
    'django.contrib.staticfiles',
    
    # Your apps
    'userauths',
    'store',
    'customer',
    'blog',
    
    # Third-party
    'django_ckeditor_5',
    'anymail',
    'captcha',
    'django_extensions',
]

# Conditionally load apps based on feature flags
# Note: Feature flags won't work at import time, so we check env vars
if env.bool('ENABLE_VENDOR', default=True):
    INSTALLED_APPS.append('vendor')

if env.bool('ENABLE_CONFIG', default=True):
    INSTALLED_APPS.insert(0, 'config')  # Load early

# ... rest of base settings
```

### ecom_prj/settings/development.py
```python
from .base import *

DEBUG = True
ALLOWED_HOSTS = ['*']

# Development database
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

# Development email backend
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```

### ecom_prj/settings/production.py
```python
from .base import *
import dj_database_url

DEBUG = False
ALLOWED_HOSTS = env.list('ALLOWED_HOSTS', default=[])

# Production database
DATABASES = {
    'default': dj_database_url.config(conn_max_age=600)
}

# Production email
EMAIL_BACKEND = 'anymail.backends.mailgun.EmailBackend'
```

## Step 7: Migration Script

### config/management/commands/init_config.py
```python
from django.core.management.base import BaseCommand
from config.models import (
    SiteConfig, FeatureFlag, ServiceFeeConfig, 
    TaxConfig, ShippingService, PaymentGateway, CurrencyConfig
)

class Command(BaseCommand):
    help = 'Initialize default configuration'

    def handle(self, *args, **options):
        # Create default site config
        if not SiteConfig.objects.exists():
            SiteConfig.objects.create(
                site_name="FastCart",
                site_tagline="Your trusted marketplace",
                contact_email="support@fastcart.com"
            )
            self.stdout.write(self.style.SUCCESS('Created default site config'))

        # Create default feature flags
        default_flags = [
            ('blog_enabled', True, 'Enable blog functionality'),
            ('multi_vendor_enabled', True, 'Enable multi-vendor marketplace'),
            ('wishlist_enabled', True, 'Enable wishlist feature'),
            ('reviews_enabled', True, 'Enable product reviews'),
            ('coupons_enabled', True, 'Enable coupon system'),
        ]
        
        for name, enabled, desc in default_flags:
            FeatureFlag.objects.get_or_create(
                name=name,
                defaults={'enabled': enabled, 'description': desc}
            )
        
        self.stdout.write(self.style.SUCCESS('Created default feature flags'))

        # Create default service fee config
        if not ServiceFeeConfig.objects.exists():
            ServiceFeeConfig.objects.create(
                fee_percentage=5.00,
                min_fee=0.00
            )
            self.stdout.write(self.style.SUCCESS('Created default service fee config'))

        # Create default tax configs
        default_taxes = [
            ('Algeria', 3),
            ('India', 6),
            ('Nigeria', 2),
            ('United States', 7),
        ]
        
        for country, rate in default_taxes:
            TaxConfig.objects.get_or_create(
                country=country,
                defaults={'tax_rate': rate}
            )
        
        self.stdout.write(self.style.SUCCESS('Created default tax configs'))

        # Create default shipping services
        default_shipping = [
            ('DHL', 'DHL', 1),
            ('FedX', 'FEDX', 2),
            ('UPS', 'UPS', 3),
            ('GIG Logistics', 'GIG', 4),
        ]
        
        for name, code, priority in default_shipping:
            ShippingService.objects.get_or_create(
                name=name,
                defaults={'code': code, 'priority': priority}
            )
        
        self.stdout.write(self.style.SUCCESS('Created default shipping services'))

        # Create default currencies
        default_currencies = [
            ('USD', '$', 'US Dollar', True, 1.0000),
            ('NGN', '₦', 'Nigerian Naira', False, 0.0012),
            ('INR', '₹', 'Indian Rupee', False, 0.012),
        ]
        
        for code, symbol, name, is_default, rate in default_currencies:
            CurrencyConfig.objects.get_or_create(
                code=code,
                defaults={
                    'symbol': symbol,
                    'name': name,
                    'is_default': is_default,
                    'exchange_rate': rate
                }
            )
        
        self.stdout.write(self.style.SUCCESS('Created default currencies'))
        self.stdout.write(self.style.SUCCESS('\nConfiguration initialized successfully!'))
```

## Step 8: Usage Examples

### In Views
```python
from config.settings_manager import is_feature_enabled, get_site_config

def index(request):
    # Check if blog is enabled
    if is_feature_enabled('blog_enabled'):
        # Show blog posts
        pass
    
    # Get site config
    site_config = get_site_config()
    context = {
        'site_name': site_config.site_name if site_config else 'FastCart',
    }
    return render(request, 'index.html', context)
```

### In Templates
```python
# In context processor (store/context.py)
from config.settings_manager import get_site_config

def default(request):
    site_config = get_site_config()
    return {
        'site_config': site_config,
        # ... other context
    }
```

```html
<!-- In templates -->
<h1>{{ site_config.site_name|default:"FastCart" }}</h1>
```

## Step 9: Run Migrations

```bash
python manage.py makemigrations config
python manage.py migrate
python manage.py init_config
```

This creates all the configuration tables and populates them with default values.

