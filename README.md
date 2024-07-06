Backend Setup (Django with MySQL)
1. Set Up a Django Project
Install Django and MySQL Client:

bash
Copy code
pip install django mysqlclient
Create a Django Project:

bash
Copy code
django-admin startproject ecommerce
cd ecommerce
Configure MySQL Database in settings.py:

python
Copy code
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'ecommerce_db',
        'USER': 'your_mysql_username',
        'PASSWORD': 'your_mysql_password',
        'HOST': 'localhost',
        'PORT': '3306',
    }
}
Create an App:

bash
Copy code
python manage.py startapp shop
Add shop to INSTALLED_APPS in settings.py.

2. Create Models
Models in shop/models.py:

python
Copy code
from django.contrib.auth.models import User
from django.db import models

class Product(models.Model):
    name = models.CharField(max_length=100)
    description = models.TextField()
    price = models.DecimalField(max_digits=10, decimal_places=2)
    stock = models.IntegerField()

    def _str_(self):
        return self.name

class Cart(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)

class CartItem(models.Model):
    cart = models.ForeignKey(Cart, on_delete=models.CASCADE)
    product = models.ForeignKey(Product, on_delete=models.CASCADE)
    quantity = models.IntegerField()

class Order(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    total = models.DecimalField(max_digits=10, decimal_places=2)

    def _str_(self):
        return f'Order {self.id}'
Migrate the Database:

bash
Copy code
python manage.py makemigrations
python manage.py migrate
3. User Authentication
Views for Registration and Authentication in shop/views.py:

python
Copy code
from django.contrib.auth import authenticate, login, logout
from django.contrib.auth.models import User
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
import json

@csrf_exempt
def register(request):
    if request.method == 'POST':
        data = json.loads(request.body)
        user = User.objects.create_user(
            username=data['username'],
            password=data['password']
        )
        return JsonResponse({'message': 'User created successfully'}, status=201)

@csrf_exempt
def user_login(request):
    if request.method == 'POST':
        data = json.loads(request.body)
        user = authenticate(username=data['username'], password=data['password'])
        if user is not None:
            login(request, user)
            return JsonResponse({'message': 'Login successful'}, status=200)
        return JsonResponse({'message': 'Invalid credentials'}, status=400)

def user_logout(request):
    logout(request)
    return JsonResponse({'message': 'Logout successful'}, status=200)
URL Patterns in shop/urls.py:

python
Copy code
from django.urls import path
from .views import register, user_login, user_logout

urlpatterns = [
    path('register/', register, name='register'),
    path('login/', user_login, name='login'),
    path('logout/', user_logout, name='logout'),
]
Include shop.urls in ecommerce/urls.py:

python
Copy code
from django.urls import path, include

urlpatterns = [
    path('shop/', include('shop.urls')),
]
4. Develop API Endpoints
API Endpoints in shop/views.py:

python
Copy code
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
from .models import Product, Cart, CartItem, Order
import json

def product_list(request):
    products = Product.objects.all().values()
    return JsonResponse(list(products), safe=False)

def product_detail(request, product_id):
    product = Product.objects.get(id=product_id)
    return JsonResponse({
        'id': product.id,
        'name': product.name,
        'description': product.description,
        'price': product.price,
        'stock': product.stock
    })

@csrf_exempt
def add_to_cart(request):
    if request.method == 'POST':
        data = json.loads(request.body)
        user = request.user
        product = Product.objects.get(id=data['product_id'])
        cart, created = Cart.objects.get_or_create(user=user)
        cart_item, created = CartItem.objects.get_or_create(cart=cart, product=product)
        cart_item.quantity += data['quantity']
        cart_item.save()
        return JsonResponse({'message': 'Product added to cart'}, status=200)

@csrf_exempt
def checkout(request):
    if request.method == 'POST':
        user = request.user
        cart = Cart.objects.get(user=user)
        order = Order.objects.create(user=user, total=sum(item.product.price * item.quantity for item in cart.cartitem_set.all()))
        cart.cartitem_set.all().delete()
        return JsonResponse({'message': 'Order created successfully', 'order_id': order.id}, status=201)
Update URL Patterns in shop/urls.py:

python
Copy code
from django.urls import path
from .views import product_list, product_detail, add_to_cart, checkout

urlpatterns += [
    path('products/', product_list, name='product_list'),
    path('products/<int:product_id>/', product_detail, name='product_detail'),
    path('cart/add/', add_to_cart, name='add_to_cart'),
    path('checkout/', checkout, name='checkout'),
]
Frontend Setup (Next.js)
1. Set Up a Next.js Project
Create a Next.js Project:

bash
Copy code
npx create-next-app ecommerce-frontend
cd ecommerce-frontend
Install Axios for API Requests:

bash
Copy code
npm install axios
2. Create Pages
User Registration and Login:

Register Page (pages/register.js):

jsx
Copy code
import { useState } from 'react';
import axios from 'axios';
import { useRouter } from 'next/router';

const Register = () => {
    const [username, setUsername] = useState('');
    const [password, setPassword] = useState('');
    const router = useRouter();

    const handleSubmit = async (e) => {
        e.preventDefault();
        await axios.post('/shop/register/', { username, password });
        router.push('/login');
    };

    return (
        <form onSubmit={handleSubmit}>
            <input type="text" value={username} onChange={(e) => setUsername(e.target.value)} placeholder="Username" />
            <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} placeholder="Password" />
            <button type="submit">Register</button>
        </form>
    );
};

export default Register;
Login Page (pages/login.js):

jsx
Copy code
import { useState } from 'react';
import axios from 'axios';
import { useRouter } from 'next/router';

const Login = () => {
    const [username, setUsername] = useState('');
    const [password, setPassword] = useState('');
    const router = useRouter();

    const handleSubmit = async (e) => {
        e.preventDefault();
        await axios.post('/shop/login/', { username, password });
        router.push('/');
    };

    return (
        <form onSubmit={handleSubmit}>
            <input type="text" value={username} onChange={(e) => setUsername(e.target.value)} placeholder="Username" />
            <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} placeholder="Password" />
            <button type="submit">Login</button>
        </form>
    );
};

export default Login;
Viewing the Product Catalogue:

Products Page (pages/index.js):
jsx
Copy code
import { useEffect, useState } from 'react';
import axios from 'axios';

const Products = () => {
    const [products, setProducts] = useState([]);

    useEffect(() => {
        const fetchProducts = async () => {
            const response = await axios.get('/shop/products/');
            setProducts(response.data);
        };
        fetchProducts();
    }, []);

    return (
        <div>
            {products.map((product) => (
                <div key={product.id}>
                    <h2>{product.name}</h2>
                    <p>{product.description}</p>
                    <p>{product.price}</p>
                </div>
            ))}
        </div>
    );
};

export default Products;
