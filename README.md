# ŞifreYenileme

🔐 Şifre Yenileme
Bu dökümanda, e-posta yoluyla şifre yenileme işleminin nasıl yapılacağını adım adım göstereceğim.

Bu uygulama, .NET Core konusunda kendini geliştirmek isteyen geliştiriciler için örnek bir projedir.

Not : Bu uygulamayı yapmak için Gereken Destekler 
Install-Package Microsoft.Extensions.Options.ConfigurationExtensions
Install-Package Microsoft.AspNetCore.Mvc
Install-Package BCrypt.Net-Next

1 
appsettings.json Ayarı 

"SMTPSettings": { //  SMTP ayarları
  "Host": "smtp.gmail.com", // SMTP sunucusu
  "Port": 587, // SMTP portu
  "Username": "smtpadmis@gmail.com", // SMTP kullanıcı adı
  "Password": "sltscsnzlqgfngnw", // SMTP şifresi : Uygulama için oluşturulmuş özel şifre
  "FromEmail": "smtpadmis@gmail.com" // SMTP gönderen e-posta adresi
}

Not : Password yazılan yer önemlidir çünkü normal mail hesabının şifresi ile buraya giriş yapamazsınız bunun yerine 
size Google 2 aşamalı bir doğrulmanın ardından 16 hanelik bir şifre verecek verilen şifreyi de boşluksuz yazmak zorundasınız 

2 
Models içersinde oluşturduğumuz Sınıf
Bu sınıf ile mail gönderme işlemimiz kolaylaşacak
namespace WebUygulama.Models
{
    public class SMTPSettings
    {
        public string Host { get; set; }
        public int Port { get; set; }
        public string Username { get; set; }
        public string Password { get; set; }
        public string FromEmail { get; set; }
    }
}

3 
Bu sınıf program.cs yüklenmesi gerekir 
builder.Services.Configure<SMTPSettings>(builder.Configuration.GetSection("SMTPSettings"));

4 Controller Bölümü 
--> Şifremi Unuttum Sayfasının Görüntüsünü Açar --> HttpGet 
[HttpGet]
public IActionResult ForgotPassword()
{
    return View();
}

--> Şifremi Unuttum Sayfasındaki Kullanıcının girdiği email doğrulama işlemi --> HttpPost 
[HttpPost]
public IActionResult ForgotPassword(string email)
{
    var kullanici = _db.Kullanici.FirstOrDefault(u => u.Email == email);
    if(kullanici == null)
    {
        TempData["Email"] = "Bu e-posta adresi ile kayıtlı bir kullanıcı bulunamadı.";
        return View();
    }
    var token = GenerateResetToken(); // Token oluşturulur
    kullanici.TokenExpireDate = DateTime.Now.AddHours(3); // Token'ın geçerlilik süresi 3 saat olarak ayarlanır 
    kullanici.ResetToken = token; // Kullanıcının ResetToken alanına token atanır
    _db.SaveChanges(); // Değişiklikler veritabanına kaydedilir
    SendResetEmail(email, token); // E-posta gönderilir
    TempData["Email"] = "Şifre sıfırlama bağlantısı e-posta adresinize gönderildi."; // Başarılı mesajı
    return RedirectToAction("ForgotPassword"); // Ana sayfaya yönlendirilir
}

--> Eski Şifreyi Yenisi ile Değiştirme Sayfası'nın Görünümü --> HttpGet 
 [HttpGet]
 public IActionResult ResetPassword(string token)
 {
     var kullanici = _db.Kullanici.FirstOrDefault(u => u.ResetToken == token && u.TokenExpireDate > DateTime.Now);
     if(kullanici == null)
     {
         TempData["Error"] = "Geçersiz veya süresi dolmuş şifre sıfırlama bağlantısı.";
         return RedirectToAction("ForgotPassword");
     }
     return View();
 }

--> Eski Şifreyi Yenisi ile Değiştirme Sayfası'nın Kullanıcı'dan aldığı bilgileri işler --> HttpPost 
[HttpPost]
public IActionResult ResetPassword(string newPassword,string token)
{
   var eskişifre = _db.Kullanici.Where(p => p.ResetToken == token).Select(p => p.Sifre).FirstOrDefault();
   bool doğrumu = ŞifreKontrol(eskişifre, newPassword);
   if(doğrumu == true)
    {
        TempData["Error"] = "Yeni şifre eski şifre ile aynı olamaz.";
        return View();
    }
    var kullanici = _db.Kullanici.FirstOrDefault(u => u.ResetToken == token && u.TokenExpireDate > DateTime.Now);
    kullanici.Sifre = HashPassword(newPassword); // Yeni şifre hash'lenir
    kullanici.ResetToken = null; // Token sıfırlanır
    kullanici.TokenExpireDate = null; // Token'ın geçerlilik süresi sıfırlanır
    _db.SaveChanges(); // Değişiklikler veritabanına kaydedilir
    TempData["Success"] = "Şifreniz başarıyla sıfırlandı."; // Başarılı mesajı
    return RedirectToAction("GirisYap"); // Giriş sayfasına yönlendirilir
}

 private bool ŞifreKontrol(string eskişifre,string yenişifre)
 {
     // Şifre kontrolü için BCrypt kullanarak eski şifre ile yeni şifreyi karşılaştır
     // Eğer True ise Eski Şifre ile yeni şifre aynı hayır değilse yeni şifre ile eski şifre bir değil
     return BCrypt.Net.BCrypt.Verify(yenişifre, eskişifre);
 }
    private string GenerateResetToken() // Kullanıcıya özel bir token oluşturmak için yaratılan metot 
   {
       // Token oluşturmak için rastgele bir dize üret
       using (var rng = new RNGCryptoServiceProvider())
       {
           var tokenData = new byte[32]; // 32 baytlık rastgele veri
           rng.GetBytes(tokenData); // Rastgele veriyi doldur
           return Convert.ToBase64String(tokenData); // Base64 formatında string olarak döndür
       }
   }

  // E-posta gönderme
private void SendResetEmail(string email, string token)
{
    // MailMessage : MailMessage, gönderilecek e-posta mesajını tanımlar.
    var mailMessage = new MailMessage
    {
        From = new MailAddress(_smtpSettings.FromEmail), // Gönderen e-posta adresi (SMTP ayarlarından alınır)
        Subject = "Şifre Sıfırlama Talebi",  // E-posta başlığı
        Body = $"Şifrenizi sıfırlamak için aşağıdaki bağlantıya tıklayın:\n" +
               $"http://localhost:5285/WithoutIdentity/ResetPassword?token={token}", // Şifre sıfırlama bağlantısı
        IsBodyHtml = false // E-posta düz metin olarak gönderilecek
    };    
    mailMessage.To.Add(email); // Alıcı e-posta adresi ekleniyor
    // SmtpClient : E posta göndermek için kullanılan sınıf
    // SMTP istemcisi oluşturuluyor ve yapılandırılıyor
    using (var smtpClient = new SmtpClient(_smtpSettings.Host)) // SMTP sunucusu (örneğin smtp.gmail.com)
    {
        smtpClient.Port = _smtpSettings.Port; // SMTP port numarası (genelde 587 veya 465)
        smtpClient.Credentials = new NetworkCredential(_smtpSettings.Username, _smtpSettings.Password); // SMTP kullanıcı adı ve şifresi
        smtpClient.EnableSsl = true; // Güvenli bağlantı (SSL/TLS) aktif ediliyor
        smtpClient.Send(mailMessage); // E-posta gönderiliyor
    }
}

  private string HashPassword(string password)
  {
      // BCrypt ile şifreyi hash'le
      return BCrypt.Net.BCrypt.HashPassword(password); // BCrypt kütüphanesi kullanılarak şifre hashlenir

  }

  Cshtmller
  Kullanıcıya göstermek istenen sayfa buralarda gösterilen ek olarak da burada işlem yapılır 

ForgotPassword.cshtml --> Şifre Sıfırlamadan önce kullanıcının email girmesi gereken sayfa
@{
    ViewData["Title"] = "Şifreyi Unuttum";
}

<h2>@ViewData["Title"]</h2>

@if (TempData["Success"] != null)
{
    <div class="alert alert-success">@TempData["Success"]</div>
}

@if (TempData["Error"] != null) // Eğer Token süresi dolmuş ise geçersiz mesajı buraya gelir 
{
    <div class="alert alert-danger">@TempData["Error"]</div>
}

<form method="post">
    <div class="form-group">
        <label for="email">E-posta adresi</label>
        <input type="email" class="form-control" id="email" name="email" required />
        @if (TempData["Email"] != null)
        {
            <span class="text-danger">@TempData["Email"]</span>
        }
    </div>
    <button type="submit" class="btn btn-primary mt-3">Şifreyi Sıfırla</button>
</form>

ResetPassword.cshtml --> Şifre Sıfırlama için doğrulaması sonrası kullanıcının gireceği yeni şifreyi belirlemek için var edildi 

@{
    ViewData["Title"] = "Şifreyi Değiştir";
}

<h2>@ViewData["Title"]</h2>

@if (TempData["Success"] != null)
{
    <div class="alert alert-success">@TempData["Success"]</div>
}

@if (TempData["Error"] != null)
{
    <div class="alert alert-danger">@TempData["Error"]</div>
}
// Onsubmit ile ek kontrol yapabiliriz kullanıcı işlemi doğru yamazsa ifade başarısız 
<form method="post" onsubmit="return Kontrol();">
    <div class="form-group">
        <label for="newPassword">Yeni Şifre</label>
        <input type="password" class="form-control" id="newPassword" name="newPassword" required />
        <p class="fw-bolder d-none"></p>
    </div>
    <div class="form-group mt-3">
        <label for="newPassword">Yeni Şifre Tekrar </label>
        <input type="password" class="form-control" id="newPassword" name="SifreTekrar" required />
    </div>
    <button type="submit" class="btn btn-primary mt-4">Şifreyi Değiştir</button>
</form>

<script>
    function Kontrol()
    {
        var şifre = document.getElementById("newPassword").value;
        var şifretekrar = document.getElementById("SifreTekrar").value;

        if(şifre != şifretekrar)
        {
            alert("Şifreler eşleşmiyor!");
            return false; // Formun gönderilmesini engeller
        }
        else 
        {
            return true; // Formun gönderilmesine izin verir
        }
    }

</script>





