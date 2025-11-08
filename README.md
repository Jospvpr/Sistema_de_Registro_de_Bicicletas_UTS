# 🚲 Sistema de Registro de Bicicletas UTS

El **Sistema de Registro de Bicicletas UTS** es una aplicación web desarrollada en **Spring Boot** que permite la **gestión y control de acceso de bicicletas** pertenecientes a estudiantes dentro del campus universitario.  
Cuenta con **autenticación y autorización** mediante **Spring Security**, integrando **roles de usuario** y **contraseñas encriptadas con BCrypt** para garantizar la seguridad de la información.

---

## 🧩 Tecnologías utilizadas

- **Java 17+**
- **Spring Boot 3+**
  - Spring MVC  
  - Spring Data JPA  
  - Spring Security  
- **Thymeleaf** (motor de plantillas HTML)
- **Bootstrap 5**
- **MySQL / MariaDB**
- **Maven**

---

## ⚙️ Funcionalidades principales

- Registro, inicio y cierre de sesión de usuarios.  
- Encriptación de contraseñas con **BCrypt**.  
- Sistema de roles (**Administrador**, **Estudiante**, **Celador**).  
- Control de permisos y acceso por rol.  
- Protección **CSRF** activa para formularios.  
- Redirección dinámica según tipo de usuario.  
- Gestión de usuarios directamente desde la base de datos.

---

## 🗂️ Estructura de la base de datos

### Tabla: `roles`
| Campo | Tipo | Descripción |
|-------|------|-------------|
| id | INT | Identificador del rol |
| nombre | VARCHAR(100) | Nombre del rol (ADMIN, CELADOR, ESTUDIANTE) |

### Tabla: `usuarios`
| Campo | Tipo | Descripción |
|-------|------|-------------|
| id | INT | Identificador del usuario |
| nombre | VARCHAR(255) | Nombre completo |
| correo | VARCHAR(255) | Correo electrónico (único) |
| contrasena | VARCHAR(255) | Contraseña cifrada con BCrypt |
| rol_id | INT | Llave foránea hacia `roles(id)` |

---

## 📦 Dependencias principales

```xml
<!-- SPRING SECURITY -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- THYMELEAF + SPRING SECURITY -->
<dependency>
    <groupId>org.thymeleaf.extras</groupId>
    <artifactId>thymeleaf-extras-springsecurity6</artifactId>
</dependency>

<!-- CIFRADO DE CONTRASEÑAS -->
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-crypto</artifactId>
</dependency>

## 🔐 Implementación de seguridad (Spring Security)

### 🔸 1. Configuración general



En la clase `SecurityConfig` se define la configuración de seguridad, los permisos de acceso y la protección CSRF:

java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/index", "/css/**", "/js/**", "/img/**").permitAll()
                .requestMatchers("/auth/login", "/auth/registro").permitAll()
                .requestMatchers("/views/estudiante/**").hasAnyRole("ESTUDIANTE", "ADMIN", "CELADOR")
                .requestMatchers("/views/bicicletas/**").hasAnyRole("ADMIN", "CELADOR")
                .requestMatchers("/views/movimientos/**").hasAnyRole("ADMIN", "CELADOR")
                .anyRequest().authenticated()
            )
            .formLogin(form -> {
                form.loginPage("/views/seguridad/login");
                form.loginProcessingUrl("/login");
                form.usernameParameter("username");
                form.passwordParameter("password");
                form.defaultSuccessUrl("/", true);
                form.failureUrl("/views/seguridad/login?error=true");
                form.permitAll();
            })
            .logout(logout -> {
                logout.logoutUrl("/logout");
                logout.logoutSuccessUrl("/views/seguridad/login?logout=true");
                logout.permitAll();
            })
            .csrf(csrf -> csrf.enable()); // Activación de protección CSRF

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}


    }
}
**Cifrado de contraseñas

El cifrado se aplica al guardar un nuevo usuario, para evitar almacenar contraseñas en texto plano:

@Override
public Usuario guardarUsuario(Usuario usuario) {
    usuario.setContrasena(passwordEncoder.encode(usuario.getContrasena()));
    return usuarioRepositorio.save(usuario);
}

**Carga de usuarios (UserDetailsService)

Spring Security utiliza una clase personalizada que implementa UserDetailsService para autenticar los usuarios desde la base de datos:

@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    @Autowired
    private UsuarioServicio usuarioServicio;

    @Override
    public UserDetails loadUserByUsername(String correo) throws UsernameNotFoundException {
        Usuario usuario = usuarioServicio.buscarPorCorreo(correo);
        if (usuario == null) {
            throw new UsernameNotFoundException("Usuario no encontrado: " + correo);
        }

        String rol = usuario.getRoles()
                .stream()
                .findFirst()
                .map(r -> r.getNombre())
                .orElse("ESTUDIANTE");

        return User.builder()
                .username(usuario.getCorreo())
                .password(usuario.getContrasena())
                .roles(rol)
                .build();
    }
}
**Registro de usuarios

El controlador maneja el registro y validación de usuarios, asignando el rol correspondiente y guardando la contraseña cifrada:

@PostMapping("/registro")
public String registrarUsuario(@ModelAttribute Usuario usuario, @RequestParam("rolId") Integer rolId, Model model) {
    if (usuarioServicio.buscarPorCorreo(usuario.getCorreo()) != null) {
        model.addAttribute("error", "Ya existe un usuario con ese correo");
        return "views/seguridad/registro";
    }

    Rol rolSeleccionado = rolServicio.buscarRol(rolId);
    usuario.setRoles(Collections.singleton(rolSeleccionado));

    usuarioServicio.guardarUsuario(usuario);
    return "views/seguridad/login";
}
