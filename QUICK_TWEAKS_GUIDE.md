# Quick Tweaks Guide: Adapting Your App for Different Projects

This guide provides quick, actionable steps to adapt your e-commerce platform for different project types.

---

## 🛍️ Project Type 1: Simple Single-Vendor Store

**Use Case:** One seller, straightforward product catalog

### Quick Changes:

1. **Disable Vendor App** (in `ecom_prj/settings.py`):
```python
# Comment out or remove:
# 'vendor',

# In INSTALLED_APPS
```

2. **Remove Vendor URLs** (in `ecom_prj/urls.py`):
```python
# Comment out:
# path('vendor/', include("vendor.urls")),
```

3. **Modify Product Model** (in `store/models.py`):
```python
# Make vendor field optional or remove vendor filtering
# Change: vendor = models.ForeignKey(...)
# To: vendor = models.ForeignKey(..., null=True, blank=True, default=None)
```

4. **Update Views** (in `store/views.py`):
```python
# Remove vendor-specific filtering
# Change product queries to not filter by vendor
```

---

## 🏪 Project Type 2: Digital Products Marketplace

**Use Case:** Software, ebooks, courses, downloads

### Quick Changes:

1. **Add Digital Product Field** (in `store/models.py`):
```python
class Product(models.Model):
    # ... existing fields ...
    is_digital = models.BooleanField(default=False)
    download_link = models.URLField(blank=True, null=True)
    download_file = models.FileField(upload_to='downloads/', blank=True, null=True)
```

2. **Disable Shipping** (in `store/models.py`):
```python
# Make shipping optional
shipping = models.DecimalField(..., default=0.00, null=True, blank=True)

# In Order model, make shipping always 0 for digital products
```

3. **Modify Checkout** (in `store/views.py`):
```python
def checkout(request):
    # Skip address collection for digital products
    # Auto-fulfill digital orders
    if product.is_digital:
        order.order_status = "Fulfilled"
        # Send download link via email
```

4. **Update Cart** (in `store/models.py`):
```python
# Set shipping to 0 for digital products in cart
```

---

## 🏢 Project Type 3: B2B Marketplace

**Use Case:** Business-to-business, bulk orders, company accounts

### Quick Changes:

1. **Add Company Model** (create `store/models.py` addition):
```python
class Company(models.Model):
    user = models.ForeignKey('userauths.User', on_delete=models.CASCADE)
    company_name = models.CharField(max_length=200)
    tax_id = models.CharField(max_length=100, blank=True)
    billing_address = models.TextField()
    
class Product(models.Model):
    # ... existing fields ...
    bulk_pricing = models.BooleanField(default=False)
    min_quantity = models.IntegerField(default=1)
    bulk_discount = models.DecimalField(max_digits=5, decimal_places=2, default=0)
```

2. **Add Bulk Pricing Logic** (in `store/views.py`):
```python
def calculate_price(product, quantity):
    if product.bulk_pricing and quantity >= product.min_quantity:
        discount = product.bulk_discount / 100
        return product.price * (1 - discount) * quantity
    return product.price * quantity
```

3. **Modify User Model** (in `userauths/models.py`):
```python
class Profile(models.Model):
    # ... existing fields ...
    is_business = models.BooleanField(default=False)
    company = models.ForeignKey('store.Company', null=True, blank=True, on_delete=models.SET_NULL)
```

---

## 📱 Project Type 4: Subscription Service

**Use Case:** Monthly/yearly subscriptions, recurring payments

### Quick Changes:

1. **Create Subscription App**:
```bash
python manage.py startapp subscriptions
```

2. **Add Subscription Models** (in `subscriptions/models.py`):
```python
from django.db import models
from store.models import Product

class SubscriptionPlan(models.Model):
    name = models.CharField(max_length=100)
    product = models.ForeignKey(Product, on_delete=models.CASCADE)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    billing_cycle = models.CharField(max_length=20, choices=[
        ('monthly', 'Monthly'),
        ('yearly', 'Yearly'),
    ])
    
class Subscription(models.Model):
    user = models.ForeignKey('userauths.User', on_delete=models.CASCADE)
    plan = models.ForeignKey(SubscriptionPlan, on_delete=models.CASCADE)
    status = models.CharField(max_length=20, choices=[
        ('active', 'Active'),
        ('cancelled', 'Cancelled'),
        ('expired', 'Expired'),
    ])
    start_date = models.DateTimeField()
    next_billing_date = models.DateTimeField()
```

3. **Modify Order Model** (in `store/models.py`):
```python
class Order(models.Model):
    # ... existing fields ...
    is_subscription = models.BooleanField(default=False)
    subscription = models.ForeignKey('subscriptions.Subscription', null=True, blank=True)
```

---

## 🎓 Project Type 5: Course/Learning Platform

**Use Case:** Online courses, tutorials, educational content

### Quick Changes:

1. **Modify Product Model** (in `store/models.py`):
```python
class Product(models.Model):
    # ... existing fields ...
    is_course = models.BooleanField(default=False)
    course_duration = models.CharField(max_length=50, blank=True)
    lessons = models.IntegerField(default=0)
    video_url = models.URLField(blank=True, null=True)
```

2. **Create Course Content Model** (in `store/models.py`):
```python
class CourseLesson(models.Model):
    course = models.ForeignKey(Product, on_delete=models.CASCADE, related_name='lessons')
    title = models.CharField(max_length=200)
    content = CKEditor5Field('Text', config_name='extends')
    video_url = models.URLField(blank=True, null=True)
    order = models.IntegerField(default=0)
    
    class Meta:
        ordering = ['order']
```

3. **Add Progress Tracking** (create `courses/models.py`):
```python
class CourseProgress(models.Model):
    user = models.ForeignKey('userauths.User', on_delete=models.CASCADE)
    course = models.ForeignKey('store.Product', on_delete=models.CASCADE)
    completed_lessons = models.ManyToManyField('store.CourseLesson', blank=True)
    progress_percentage = models.IntegerField(default=0)
```

---

## 🌍 Project Type 6: Multi-Currency International Store

**Use Case:** Selling globally with multiple currencies

### Quick Changes:

1. **Add Currency Field** (in `store/models.py`):
```python
class Product(models.Model):
    # ... existing fields ...
    currency = models.CharField(max_length=3, default='USD', choices=[
        ('USD', 'US Dollar'),
        ('EUR', 'Euro'),
        ('NGN', 'Nigerian Naira'),
        ('INR', 'Indian Rupee'),
    ])
```

2. **Update Exchange Rate Plugin** (in `plugin/exchange_rate.py`):
```python
# Already exists, but enhance it:
def convert_currency(amount, from_currency, to_currency):
    if from_currency == to_currency:
        return amount
    
    # Fetch rates and convert
    rates = fetch_exchange_rates()
    # Add conversion logic
```

3. **Modify Cart/Order** (in `store/models.py`):
```python
class Cart(models.Model):
    # ... existing fields ...
    currency = models.CharField(max_length=3, default='USD')
    
class Order(models.Model):
    # ... existing fields ...
    currency = models.CharField(max_length=3, default='USD')
```

---

## 🎨 Project Type 7: Customizable Product Store

**Use Case:** Print-on-demand, custom designs, personalized products

### Quick Changes:

1. **Add Customization Options** (in `store/models.py`):
```python
class ProductCustomization(models.Model):
    product = models.ForeignKey(Product, on_delete=models.CASCADE)
    option_name = models.CharField(max_length=100)  # e.g., "Text", "Color", "Size"
    option_type = models.CharField(max_length=20, choices=[
        ('text', 'Text Input'),
        ('color', 'Color Picker'),
        ('image', 'Image Upload'),
        ('dropdown', 'Dropdown'),
    ])
    required = models.BooleanField(default=False)
    price_adjustment = models.DecimalField(max_digits=10, decimal_places=2, default=0)

class OrderItemCustomization(models.Model):
    order_item = models.ForeignKey('store.OrderItem', on_delete=models.CASCADE)
    customization = models.ForeignKey(ProductCustomization, on_delete=models.CASCADE)
    value = models.TextField()  # Stores the customization value
```

2. **Update Cart Model** (in `store/models.py`):
```python
class Cart(models.Model):
    # ... existing fields ...
    customizations = models.JSONField(default=dict, blank=True)  # Store customization data
```

---

## 🚀 Project Type 8: Auction/Marketplace

**Use Case:** Bidding, auctions, time-limited sales

### Quick Changes:

1. **Add Auction Fields** (in `store/models.py`):
```python
class Product(models.Model):
    # ... existing fields ...
    is_auction = models.BooleanField(default=False)
    auction_start = models.DateTimeField(null=True, blank=True)
    auction_end = models.DateTimeField(null=True, blank=True)
    starting_bid = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    current_bid = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    bid_increment = models.DecimalField(max_digits=10, decimal_places=2, default=1)

class Bid(models.Model):
    product = models.ForeignKey(Product, on_delete=models.CASCADE)
    user = models.ForeignKey('userauths.User', on_delete=models.CASCADE)
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    timestamp = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        ordering = ['-amount', '-timestamp']
```

2. **Add Auction Views** (in `store/views.py`):
```python
def place_bid(request, product_id):
    product = get_object_or_404(Product, id=product_id)
    if request.method == 'POST':
        bid_amount = Decimal(request.POST.get('bid_amount'))
        # Validate bid
        if bid_amount > product.current_bid + product.bid_increment:
            Bid.objects.create(
                product=product,
                user=request.user,
                amount=bid_amount
            )
            product.current_bid = bid_amount
            product.save()
```

---

## 🔧 Common Tweaks for All Projects

### 1. Change Site Name/Branding

**Quick Fix:**
```python
# In ecom_prj/settings.py
JAZZMIN_SETTINGS = {
    "site_title": "Your Store Name",
    "site_header": "Your Store Name",
    "site_brand": "Your Store Name",
    # ...
}
```

### 2. Disable Blog

**Quick Fix:**
```python
# In ecom_prj/settings.py - INSTALLED_APPS
# Comment out: 'blog',

# In ecom_prj/urls.py
# Comment out: path('blog/', include("blog.urls")),
```

### 3. Change Default Currency

**Quick Fix:**
```python
# In store/models.py - Update default currency in Product model
currency = models.CharField(max_length=3, default='NGN')  # or EUR, INR, etc.
```

### 4. Modify Payment Gateways

**Quick Fix:**
```python
# In store/models.py - PAYMENT_METHOD choices
PAYMENT_METHOD = (
    ("PayPal", "PayPal"),
    ("Stripe", "Stripe"),
    # Remove or comment out gateways you don't need
    # ("Flutterwave", "Flutterwave"),
    # ("Paystack", "Paystack"),
    # ("RazorPay", "RazorPay"),
)
```

### 5. Change Service Fee

**Quick Fix:**
```python
# In plugin/service_fee.py
def calculate_service_fee(order_total):
    service_fee = 3  # Change from 5 to 3, or any percentage
    return Decimal(order_total) * Decimal(service_fee) / 100
```

### 6. Add/Remove Countries

**Quick Fix:**
```python
# In plugin/countries.py
def countries():
    return [
        {"country": "Your Country", "tax_rate": "5"},  # Add new
        # Remove countries you don't need
    ]
```

### 7. Change Shipping Services

**Quick Fix:**
```python
# In store/models.py - SHIPPING_SERVICE choices
SHIPPING_SERVICE = (
    ("DHL", "DHL"),
    ("Your Local Service", "Your Local Service"),  # Add new
    # Remove services you don't use
)
```

### 8. Disable Reviews

**Quick Fix:**
```python
# In store/views.py - product_detail view
# Comment out review-related code

# In templates - Remove review section
```

### 9. Change Order Status Options

**Quick Fix:**
```python
# In store/models.py - ORDER_STATUS choices
ORDER_STATUS = (
    ("Pending", "Pending"),
    ("Processing", "Processing"),
    ("Shipped", "Shipped"),
    ("Delivered", "Delivered"),  # Change "Fulfilled" to "Delivered"
    ("Cancelled", "Cancelled"),
)
```

### 10. Modify User Types

**Quick Fix:**
```python
# In userauths/models.py - USER_TYPE choices
USER_TYPE = (
    ("Vendor", "Vendor"),
    ("Customer", "Customer"),
    ("Admin", "Admin"),  # Add new type
)
```

---

## 📋 Checklist for New Project Setup

- [ ] Update site name/branding in settings
- [ ] Configure payment gateways (enable/disable)
- [ ] Set default currency
- [ ] Configure tax rates for target countries
- [ ] Set up shipping services
- [ ] Adjust service fee percentage
- [ ] Enable/disable features (blog, reviews, wishlist)
- [ ] Update email templates
- [ ] Configure SMTP/email backend
- [ ] Set up database (SQLite for dev, PostgreSQL for prod)
- [ ] Configure static/media file storage
- [ ] Update allowed hosts
- [ ] Set up environment variables
- [ ] Run migrations
- [ ] Create superuser
- [ ] Test payment gateways
- [ ] Test order flow
- [ ] Test email notifications

---

## 🎯 Quick Environment Setup

### Development
```bash
# .env file
DEBUG=True
SECRET_KEY=your-dev-secret-key
DATABASE_URL=sqlite:///db.sqlite3
EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
```

### Production
```bash
# .env file
DEBUG=False
SECRET_KEY=your-production-secret-key
DATABASE_URL=postgresql://user:pass@host:port/db
EMAIL_BACKEND=anymail.backends.mailgun.EmailBackend
ALLOWED_HOSTS=yourdomain.com,www.yourdomain.com
```

---

## 💡 Pro Tips

1. **Use Environment Variables**: Never hardcode sensitive data
2. **Feature Flags**: Use feature flags for easy enable/disable
3. **Database Migrations**: Always test migrations on a copy first
4. **Backup Before Changes**: Always backup database before major changes
5. **Version Control**: Commit changes incrementally
6. **Testing**: Test payment flows in sandbox/test mode first
7. **Documentation**: Document any custom changes you make

---

*This guide provides quick tweaks. For more comprehensive customization, refer to APP_ASSESSMENT.md and IMPLEMENTATION_EXAMPLE.md*

