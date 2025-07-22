# Simple-E-Commerce-API
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;
    private String username;
    private String password;
    private String role; // "ROLE_CUSTOMER" or "ROLE_ADMIN"
}



@Entity
public class Product {
    @Id @GeneratedValue
    private Long id;
    private String name;
    private double price;
    private S tring description;
}



@Entity
public class CartItem {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne
    private User user;

    @ManyToOne
    private Product product;

    private int quantity;
}



@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne
    private User user;

    private LocalDateTime orderDate;

    @OneToMany
    private List<CartItem> items;





@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired private JwtFilter jwtFilter;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable()
            .authorizeRequests()
            .antMatchers("/auth/**").permitAll()
            .antMatchers(HttpMethod.POST, "/products").hasRole("ADMIN")
            .antMatchers(HttpMethod.PUT, "/products/**").hasRole("ADMIN")
            .antMatchers(HttpMethod.DELETE, "/products/**").hasRole("ADMIN")
            .anyRequest().authenticated()
            .and().sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);

        http.addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
    }
}









@RestController
@RequestMapping("/products")
public class ProductController {
    @Autowired private ProductService productService;

    @GetMapping
    public List<Product> getAllProducts() {
        return productService.getAll();
    }

    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public Product addProduct(@RequestBody Product product) {
        return productService.save(product);
    }

    @PutMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public Product update(@PathVariable Long id, @RequestBody Product p) {
        return productService.update(id, p);
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public void delete(@PathVariable Long id) {
        productService.delete(id);
    }
}






@RestController
@RequestMapping("/cart")
public class CartController {
    @Autowired private CartService cartService;

    @PostMapping("/add")
    public ResponseEntity<?> addToCart(@RequestBody CartItem item, Principal principal) {
        cartService.addToCart(item, principal.getName());
        return ResponseEntity.ok("Added to cart");
    }

    @GetMapping
    public List<CartItem> getCart(Principal principal) {
        return cartService.getCart(principal.getName());
    }

    @DeleteMapping("/{id}")
    public void removeItem(@PathVariable Long id) {
        cartService.removeItem(id);
    }
}






@RestController
@RequestMapping("/orders")
public class OrderController {
    @Autowired private OrderService orderService;

    @PostMapping("/place")
    public ResponseEntity<?> placeOrder(Principal principal) {
        orderService.placeOrder(principal.getName());
        return ResponseEntity.ok("Order placed!");
    }

    @GetMapping
    public List<Order> getMyOrders(Principal principal) {
        return orderService.getOrdersByUser(principal.getName());
    }
}}

//Product Pagination and Search
public List<Product> getAllProducts(int page, int size, String name, String category) {
    Pageable pageable = PageRequest.of(page, size);
    if (name != null) {
        return productRepo.findByNameContainingIgnoreCase(name, pageable);
    } else if (category != null) {
        return productRepo.findByCategoryIgnoreCase(category, pageable);
    } else {
        return productRepo.findAll(pageable).getContent();
    }
}

//Basic Frontend HTML Page that intercts with a Backend E-commerce API 
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>E-commerce Frontend</title>
  <style>
    body { font-family: Arial; padding: 20px; }
    button, input { margin: 5px; }
    .section { margin-bottom: 30px; }
    pre { background: #f0f0f0; padding: 10px; }
  </style>
</head>
<body>

  <h1>Simple E-commerce Frontend</h1>

  <!-- Login -->
  <div class="section">
    <h2>Login</h2>
    <input type="text" id="username" placeholder="Username">
    <input type="password" id="password" placeholder="Password">
    <button onclick="login()">Login</button>
  </div>

  <!-- Product List -->
  <div class="section">
    <h2>Products</h2>
    <button onclick="fetchProducts()">Load Products</button>
    <pre id="products"></pre>
  </div>

  <!-- Add to Cart -->
  <div class="section">
    <h2>Add to Cart</h2>
    <input type="text" id="productId" placeholder="Product ID">
    <input type="number" id="quantity" placeholder="Quantity">
    <button onclick="addToCart()">Add to Cart</button>
  </div>

  <!-- View Cart -->
  <div class="section">
    <h2>Cart</h2>
    <button onclick="viewCart()">View Cart</button>
    <pre id="cart"></pre>
  </div>

  <!-- Place Order -->
  <div class="section">
    <h2>Order</h2>
    <button onclick="placeOrder()">Place Order</button>
    <pre id="orderResponse"></pre>
  </div>

  <script>
    let token = '';

    async function login() {
      const res = await fetch('http://localhost:8080/api/auth/login', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({
          username: document.getElementById('username').value,
          password: document.getElementById('password').value
        })
      });
      const data = await res.json();
      if (data.token) {
        token = data.token;
        alert('Login successful!');
      } else {
        alert('Login failed');
      }
    }

    async function fetchProducts() {
      const res = await fetch('http://localhost:8080/api/products');
      const data = await res.json();
      document.getElementById('products').textContent = JSON.stringify(data, null, 2);
    }

    async function addToCart() {
      const res = await fetch('http://localhost:8080/api/cart/add', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': 'Bearer ' + token
        },
        body: JSON.stringify({
          productId: document.getElementById('productId').value,
          quantity: parseInt(document.getElementById('quantity').value)
        })
      });
      const data = await res.json();
      alert('Item added to cart');
    }

    async function viewCart() {
      const res = await fetch('http://localhost:8080/api/cart', {
        headers: { 'Authorization': 'Bearer ' + token }
      });
      const data = await res.json();
      document.getElementById('cart').textContent = JSON.stringify(data, null, 2);
    }

    async function placeOrder() {
      const res = await fetch('http://localhost:8080/api/orders', {
        method: 'POST',
        headers: { 'Authorization': 'Bearer ' + token }
      });
      const data = await res.json();
      document.getElementById('orderResponse').textContent = JSON.stringify(data, null, 2);
    }
  </script>

</body>
</html>



