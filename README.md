# roundrapcompany
=== accounts/models.py ===
from django.db import models
from django.contrib.auth.models import AbstractUser
from django.core.validators import MinValueValidator

class User(AbstractUser):
    USER_TYPE_CHOICES = (
        ('buyer', '구매자'),
        ('seller_business', '사업자 셀러'),
        ('seller_freelancer', 'P2P 프리랜서 셀러'),
    )
    
    STATUS_CHOICES = (
        ('pending', '대기중'),
        ('approved', '승인됨'),
        ('rejected', '거절됨'),
        ('suspended', '정지됨'),
        ('active', '활성화'),
    )
    
    user_type = models.CharField(max_length=20, choices=USER_TYPE_CHOICES, default='buyer')
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='pending')
    phone = models.CharField(max_length=11, blank=True)
    company_name = models.CharField(max_length=255, blank=True, null=True)
    business_number = models.CharField(max_length=20, blank=True, null=True, unique=True)
    representative_name = models.CharField(max_length=100, blank=True, null=True)
    business_address = models.CharField(max_length=500, blank=True, null=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    def __str__(self):
        return f"{self.username} ({self.get_user_type_display()})"
    
    class Meta:
        verbose_name = '사용자'
        verbose_name_plural = '사용자들'


class PointAccount(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='point_account')
    balance = models.DecimalField(max_digits=10, decimal_places=2, default=0, validators=[MinValueValidator(0)])
    total_charged = models.DecimalField(max_digits=12, decimal_places=2, default=0)
    total_withdrawn = models.DecimalField(max_digits=12, decimal_places=2, default=0)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    def __str__(self):
        return f"{self.user.username} - ₩{self.balance:,.0f}"
    
    class Meta:
        verbose_name = '포인트 계정'
        verbose_name_plural = '포인트 계정들'


class Bank(models.Model):
    bank_name = models.CharField(max_length=50)
    account_number = models.CharField(max_length=30, unique=True)
    account_holder = models.CharField(max_length=100)
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return f"{self.bank_name} - {self.account_number} ({self.account_holder})"
    
    class Meta:
        verbose_name = '계좌'
        verbose_name_plural = '계좌들'


class UserBankAccount(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='bank_account')
    bank_name = models.CharField(max_length=50)
    account_number = models.CharField(max_length=30)
    account_holder = models.CharField(max_length=100)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    def __str__(self):
        return f"{self.user.username} - {self.bank_name} {self.account_number}"
    
    class Meta:
        verbose_name = '사용자 계좌'
        verbose_name_plural = '사용자 계좌들'


=== accounts/admin.py ===
from django.contrib import admin
from .models import User, PointAccount, Bank, UserBankAccount

@admin.register(User)
class UserAdmin(admin.ModelAdmin):
    list_display = ('username', 'email', 'user_type', 'status', 'created_at')
    list_filter = ('user_type', 'status', 'created_at')
    search_fields = ('username', 'email', 'business_number')
    fieldsets = (
        ('기본정보', {'fields': ('username', 'email', 'password')}),
        ('개인정보', {'fields': ('first_name', 'last_name', 'phone')}),
        ('회원유형', {'fields': ('user_type', 'status')}),
        ('사업자정보', {'fields': ('company_name', 'business_number', 'representative_name', 'business_address')}),
        ('권한', {'fields': ('is_active', 'is_staff', 'is_superuser', 'groups', 'user_permissions')}),
        ('기타', {'fields': ('last_login', 'date_joined')}),
    )

@admin.register(PointAccount)
class PointAccountAdmin(admin.ModelAdmin):
    list_display = ('user', 'balance', 'total_charged', 'total_withdrawn')
    search_fields = ('user__username',)
    readonly_fields = ('created_at', 'updated_at')

@admin.register(Bank)
class BankAdmin(admin.ModelAdmin):
    list_display = ('bank_name', 'account_number', 'account_holder', 'is_active')
    list_filter = ('is_active',)

@admin.register(UserBankAccount)
class UserBankAccountAdmin(admin.ModelAdmin):
    list_display = ('user', 'bank_name', 'account_number', 'account_holder', 'created_at')
    search_fields = ('user__username',)
    readonly_fields = ('created_at', 'updated_at')


=== accounts/views.py ===
from django.shortcuts import render, redirect
from django.contrib.auth import authenticate, login, logout
from django.contrib.auth.decorators import login_required
from django.views.decorators.http import require_http_methods
from django.contrib import messages
from .models import User, PointAccount, UserBankAccount
from .forms import SignUpForm, LoginForm
from points.models import PointHistory

@require_http_methods(["GET", "POST"])
def signup(request):
    if request.method == 'POST':
        form = SignUpForm(request.POST)
        if form.is_valid():
            user = form.save(commit=False)
            user.status = 'pending'
            user.save()
            PointAccount.objects.create(user=user)
            messages.success(request, '회원가입이 완료되었습니다. 관리자의 승인을 기다려주세요.')
            return redirect('login')
    else:
        form = SignUpForm()
    return render(request, 'accounts/signup.html', {'form': form})


@require_http_methods(["GET", "POST"])
def login_view(request):
    if request.user.is_authenticated:
        return redirect('product_list')
    
    if request.method == 'POST':
        form = LoginForm(request.POST)
        if form.is_valid():
            username = form.cleaned_data['username']
            password = form.cleaned_data['password']
            user = authenticate(request, username=username, password=password)
            
            if user is not None:
                if user.status == 'approved':
                    login(request, user)
                    return redirect('product_list')
                elif user.status == 'pending':
                    messages.error(request, '관리자의 승인을 기다리고 있습니다.')
                elif user.status == 'rejected':
                    messages.error(request, '승인이 거절되었습니다.')
                elif user.status == 'suspended':
                    messages.error(request, '계정이 정지되었습니다.')
            else:
                messages.error(request, '아이디 또는 비밀번호가 잘못되었습니다.')
    else:
        form = LoginForm()
    
    return render(request, 'accounts/login.html', {'form': form})


def logout_view(request):
    logout(request)
    messages.success(request, '로그아웃되었습니다.')
    return redirect('login')


@login_required
@require_http_methods(["GET", "POST"])
def mypage(request):
    point_account = request.user.point_account
    recent_history = PointHistory.objects.filter(user=request.user).order_by('-created_at')[:10]
    
    try:
        user_bank_account = request.user.bank_account
    except:
        user_bank_account = None
    
    if request.method == 'POST':
        bank_name = request.POST.get('bank_name')
        account_number = request.POST.get('account_number')
        account_holder = request.POST.get('account_holder')
        
        if bank_name and account_number and account_holder:
            UserBankAccount.objects.update_or_create(
                user=request.user,
                defaults={
                    'bank_name': bank_name,
                    'account_number': account_number,
                    'account_holder': account_holder,
                }
            )
            messages.success(request, '계좌정보가 저장되었습니다.')
            return redirect('mypage')
        else:
            messages.error(request, '모든 항목을 입력해주세요.')
    
    context = {
        'point_account': point_account,
        'recent_history': recent_history,
        'user_bank_account': user_bank_account,
    }
    return render(request, 'accounts/mypage.html', context)


=== points/views.py (point_withdraw 함수만 수정) ===
@login_required
@require_http_methods(["GET", "POST"])
def point_withdraw(request):
    point_account = request.user.point_account
    settings = SystemSettings.objects.first()
    
    try:
        user_bank_account = request.user.bank_account
    except:
        user_bank_account = None
    
    if request.method == 'POST':
        amount = Decimal(request.POST.get('amount', 0))
        bank_name = request.POST.get('bank_name')
        account_number = request.POST.get('account_number')
        account_holder = request.POST.get('account_holder')
        
        if amount <= 0:
            messages.error(request, '환전 금액은 0보다 커야 합니다.')
            return redirect('point_withdraw')
        
        if point_account.balance < amount:
            messages.error(request, '포인트 잔액이 부족합니다.')
            return redirect('point_withdraw')
        
        fee = Decimal(0)
        if settings and settings.withdraw_fee_enabled:
            fee = amount * settings.withdraw_fee_rate / 100
        
        withdraw_request = WithdrawRequest.objects.create(
            user=request.user,
            amount=amount,
            fee=fee,
            bank_name=bank_name,
            account_number=account_number,
            account_holder=account_holder,
        )
        
        messages.success(request, '환전 신청이 완료되었습니다.')
        return redirect('mypage')
    
    context = {
        'point_account': point_account,
        'settings': settings,
        'user_bank_account': user_bank_account,
    }
    return render(request, 'points/point_withdraw.html', context)


=== templates/accounts/mypage.html ===
{% extends 'base.html' %}

{% block title %}마이페이지 - 주식회사 라운드{% endblock %}

{% block content %}
<div class="container mt-5 mb-5">
    <h1 class="mb-4"><i class="bi bi-person"></i> 마이페이지</h1>
    
    <div class="row mb-4">
        <div class="col-md-3">
            <div class="stat-box">
                <div class="stat-label">포인트 잔액</div>
                <div class="price-display">₩{{ point_account.balance|floatformat:0 }}</div>
                <small>누적 충전: ₩{{ point_account.total_charged|floatformat:0 }}</small>
            </div>
        </div>
        <div class="col-md-3">
            <div class="stat-box">
                <div class="stat-label">누적 환전</div>
                <div class="price-display">₩{{ point_account.total_withdrawn|floatformat:0 }}</div>
            </div>
        </div>
        <div class="col-md-6">
            <div class="card h-100">
                <div class="card-body">
                    <h5 class="card-title">빠른 메뉴</h5>
                    <a href="{% url 'point_charge' %}" class="btn btn-primary btn-sm w-100 mb-2">
                        <i class="bi bi-plus-circle"></i> 포인트 충전
                    </a>
                    <a href="{% url 'point_withdraw' %}" class="btn btn-outline-primary btn-sm w-100">
                        <i class="bi bi-dash-circle"></i> 포인트 환전
                    </a>
                </div>
            </div>
        </div>
    </div>

    <div class="card mb-4">
        <div class="card-header" style="background: linear-gradient(135deg, var(--secondary-color) 0%, var(--primary-color) 100%); color: white; border: none;">
            <h5 class="mb-0"><i class="bi bi-bank"></i> 환전 계좌 정보</h5>
        </div>
        <div class="card-body">
            <form method="post">
                {% csrf_token %}
                <div class="row">
                    <div class="col-md-3">
                        <label class="form-label fw-bold">입금받을 은행</label>
                        <select name="bank_name" class="form-select" required>
                            <option value="">은행을 선택해주세요</option>
                            <option value="국민은행" {% if user_bank_account.bank_name == '국민은행' %}selected{% endif %}>국민은행</option>
                            <option value="신한은행" {% if user_bank_account.bank_name == '신한은행' %}selected{% endif %}>신한은행</option>
                            <option value="하나은행" {% if user_bank_account.bank_name == '하나은행' %}selected{% endif %}>하나은행</option>
                            <option value="우리은행" {% if user_bank_account.bank_name == '우리은행' %}selected{% endif %}>우리은행</option>
                            <option value="농협" {% if user_bank_account.bank_name == '농협' %}selected{% endif %}>농협</option>
                            <option value="기업은행" {% if user_bank_account.bank_name == '기업은행' %}selected{% endif %}>기업은행</option>
                            <option value="SC제일은행" {% if user_bank_account.bank_name == 'SC제일은행' %}selected{% endif %}>SC제일은행</option>
                            <option value="대출은행" {% if user_bank_account.bank_name == '대출은행' %}selected{% endif %}>대출은행</option>
                        </select>
                    </div>
                    <div class="col-md-3">
                        <label class="form-label fw-bold">계좌번호</label>
                        <input type="text" name="account_number" class="form-control" placeholder="계좌번호" value="{% if user_bank_account %}{{ user_bank_account.account_number }}{% endif %}" required>
                    </div>
                    <div class="col-md-3">
                        <label class="form-label fw-bold">예금주명</label>
                        <input type="text" name="account_holder" class="form-control" placeholder="예금주명" value="{% if user_bank_account %}{{ user_bank_account.account_holder }}{% endif %}" required>
                    </div>
                    <div class="col-md-3 d-flex align-items-end">
                        <button type="submit" class="btn btn-primary w-100 fw-bold">
                            <i class="bi bi-check-circle"></i> 저장
                        </button>
                    </div>
                </div>
                {% if user_bank_account %}
                <small class="text-muted d-block mt-2">
                    <i class="bi bi-info-circle"></i> 마지막 수정: {{ user_bank_account.updated_at|date:"Y-m-d H:i" }}
                </small>
                {% endif %}
            </form>
        </div>
    </div>
    
    <div class="card">
        <div class="card-header">
            <h5 class="mb-0"><i class="bi bi-clock-history"></i> 최근 거래 내역</h5>
        </div>
        <div class="table-responsive">
            <table class="table">
                <thead>
                    <tr>
                        <th>날짜</th>
                        <th>거래 유형</th>
                        <th>금액</th>
                        <th>수수료</th>
                        <th>상태</th>
                    </tr>
                </thead>
                <tbody>
                    {% for history in recent_history %}
                    <tr>
                        <td>{{ history.created_at|date:"Y-m-d H:i" }}</td>
                        <td>{{ history.get_transaction_type_display }}</td>
                        <td>₩{{ history.amount|floatformat:0 }}</td>
                        <td>₩{{ history.fee|floatformat:0 }}</td>
                        <td><span class="badge bg-success">{{ history.get_status_display }}</span></td>
                    </tr>
                    {% empty %}
                    <tr>
                        <td colspan="5" class="text-center text-muted py-4">거래 내역이 없습니다.</td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
    </div>
</div>
{% endblock %}


=== templates/points/point_withdraw.html ===
{% extends 'base.html' %}

{% block title %}포인트 환전 - 주식회사 라운드{% endblock %}

{% block content %}
<div class="container mt-5 mb-5">
    <div class="row justify-content-center">
        <div class="col-md-7">
            <div class="card">
                <div class="card-header" style="background: linear-gradient(135deg, var(--secondary-color) 0%, var(--primary-color) 100%); color: white; border: none;">
                    <h3 class="mb-0"><i class="bi bi-dash-circle"></i> 포인트 환전</h3>
                </div>
                <div class="card-body p-5">
                    <div class="alert alert-info mb-4">
                        <strong>현재 포인트 잔액:</strong> <span class="price-display">₩{{ point_account.balance|floatformat:0 }}</span>
                    </div>
                    
                    <form method="post">
                        {% csrf_token %}
                        
                        <div class="mb-4">
                            <label for="amount" class="form-label fw-bold">환전 금액</label>
                            <div class="input-group">
                                <input type="number" name="amount" id="amount" class="form-control" placeholder="환전 금액 입력" min="1000" step="1000" max="{{ point_account.balance|floatformat:0 }}" required>
                                <span class="input-group-text">원</span>
                            </div>
                            <small class="text-muted">최소 1,000원 이상 환전 가능합니다.</small>
                        </div>
                        
                        {% if settings.withdraw_fee_enabled %}
                        <div class="alert alert-warning" role="alert">
                            <i class="bi bi-exclamation-circle"></i> <strong>수수료 안내</strong><br>
                            환전 금액의 <strong>{{ settings.withdraw_fee_rate }}%</strong> 수수료가 발생합니다.
                        </div>
                        {% endif %}
                        
                        <div class="mb-4">
                            <label for="bank_name" class="form-label fw-bold">입금받을 은행</label>
                            <select name="bank_name" id="bank_name" class="form-select" required>
                                <option value="">은행을 선택해주세요</option>
                                <option value="국민은행" {% if user_bank_account.bank_name == '국민은행' %}selected{% endif %}>국민은행</option>
                                <option value="신한은행" {% if user_bank_account.bank_name == '신한은행' %}selected{% endif %}>신한은행</option>
                                <option value="하나은행" {% if user_bank_account.bank_name == '하나은행' %}selected{% endif %}>하나은행</option>
                                <option value="우리은행" {% if user_bank_account.bank_name == '우리은행' %}selected{% endif %}>우리은행</option>
                                <option value="농협" {% if user_bank_account.bank_name == '농협' %}selected{% endif %}>농협</option>
                                <option value="기업은행" {% if user_bank_account.bank_name == '기업은행' %}selected{% endif %}>기업은행</option>
                                <option value="SC제일은행" {% if user_bank_account.bank_name == 'SC제일은행' %}selected{% endif %}>SC제일은행</option>
                                <option value="대출은행" {% if user_bank_account.bank_name == '대출은행' %}selected{% endif %}>대출은행</option>
                            </select>
                        </div>
                        
                        <div class="mb-4">
                            <label for="account_number" class="form-label fw-bold">계좌번호</label>
                            <input type="text" name="account_number" id="account_number" class="form-control" placeholder="계좌번호" value="{% if user_bank_account %}{{ user_bank_account.account_number }}{% endif %}" required>
                        </div>
                        
                        <div class="mb-4">
                            <label for="account_holder" class="form-label fw-bold">예금주명</label>
                            <input type="text" name="account_holder" id="account_holder" class="form-control" placeholder="예금주명" value="{% if user_bank_account %}{{ user_bank_account.account_holder }}{% endif %}" required>
                        </div>
                        
                        <button type="submit" class="btn btn-primary w-100 py-3 fw-bold">
                            <i class="bi bi-check-circle"></i> 환전 신청
                        </button>
                    </form>
                </div>
            </div>
        </div>
    </div>
</div>
{% endblock %}


=== 터미널 명령어 ===
python manage.py makemigrations accounts
python manage.py migrate
