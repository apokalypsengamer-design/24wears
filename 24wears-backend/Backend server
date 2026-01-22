const express = require('express');
const cors = require('cors');
const nodemailer = require('nodemailer');
const fetch = require('node-fetch');
require('dotenv').config();

const app = express();
const PORT = process.env.PORT || 3000;

app.use(cors());
app.use(express.json());

const paypal = require('@paypal/checkout-server-sdk');

function environment() {
  const clientId = process.env.PAYPAL_CLIENT_ID;
  const clientSecret = process.env.PAYPAL_CLIENT_SECRET;
  
  if (process.env.NODE_ENV === 'production') {
    return new paypal.core.LiveEnvironment(clientId, clientSecret);
  } else {
    return new paypal.core.SandboxEnvironment(clientId, clientSecret);
  }
}

const client = new paypal.core.PayPalHttpClient(environment());

const transporter = nodemailer.createTransport({
  host: process.env.SMTP_HOST || 'smtp.gmail.com',
  port: process.env.SMTP_PORT || 587,
  secure: false,
  auth: {
    user: process.env.EMAIL_USER,
    pass: process.env.EMAIL_PASSWORD
  }
});

app.get('/api/health', (req, res) => {
  res.json({ 
    status: 'ok', 
    environment: process.env.NODE_ENV || 'development',
    timestamp: new Date().toISOString()
  });
});

app.get('/api/config/paypal', (req, res) => {
  res.json({
    clientId: process.env.PAYPAL_CLIENT_ID,
    currency: 'EUR'
  });
});

app.post('/api/orders', async (req, res) => {
  try {
    const { customer, items, shipping } = req.body;

    if (!customer || !items || !Array.isArray(items) || items.length === 0) {
      return res.status(400).json({ 
        error: 'Invalid order data',
        message: 'Customer information and items are required'
      });
    }

    const subtotal = items.reduce((sum, item) => sum + (item.price * (item.quantity || 1)), 0);
    const shippingCost = subtotal >= 50 ? 0 : 3.49;
    const total = subtotal + shippingCost;

    const orderId = `24W-${Date.now()}`;
    const orderData = {
      orderId,
      customer,
      items,
      subtotal: subtotal.toFixed(2),
      shipping: shippingCost.toFixed(2),
      total: total.toFixed(2),
      status: 'pending',
      createdAt: new Date().toISOString()
    };

    await sendDiscordNotification(orderData);

    res.status(201).json({
      success: true,
      orderId: orderData.orderId,
      total: orderData.total
    });

  } catch (error) {
    console.error('Order creation error:', error);
    res.status(500).json({ 
      error: 'Order creation failed',
      message: error.message 
    });
  }
});

app.post('/api/orders/:orderId/confirm', async (req, res) => {
  try {
    const { orderId } = req.params;
    const { paypalDetails } = req.body;

    if (!paypalDetails || !paypalDetails.id) {
      return res.status(400).json({ 
        error: 'Invalid payment details' 
      });
    }

    const request = new paypal.orders.OrdersGetRequest(paypalDetails.id);
    const order = await client.execute(request);

    if (order.result.status !== 'COMPLETED') {
      return res.status(400).json({ 
        error: 'Payment not completed' 
      });
    }

    const confirmationData = {
      orderId,
      paypalTransactionId: paypalDetails.id,
      status: 'paid',
      paidAt: new Date().toISOString()
    };

    await sendDiscordOrderConfirmation(confirmationData);
    await sendCustomerEmail(confirmationData);

    res.json({
      success: true,
      orderId,
      transactionId: paypalDetails.id
    });

  } catch (error) {
    console.error('Order confirmation error:', error);
    res.status(500).json({ 
      error: 'Order confirmation failed',
      message: error.message 
    });
  }
});

app.post('/api/newsletter/subscribe', async (req, res) => {
  try {
    const { email } = req.body;

    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!email || !emailRegex.test(email)) {
      return res.status(400).json({ 
        error: 'Invalid email address' 
      });
    }

    await sendDiscordNewsletterNotification(email);

    res.json({
      success: true,
      message: 'Successfully subscribed to newsletter'
    });

  } catch (error) {
    console.error('Newsletter subscription error:', error);
    res.status(500).json({ 
      error: 'Subscription failed',
      message: error.message 
    });
  }
});

async function sendDiscordNotification(orderData) {
  const webhookUrl = process.env.DISCORD_WEBHOOK_ORDERS;
  
  if (!webhookUrl) return;

  const itemsList = orderData.items.map(item => 
    `• ${item.name} (${item.selectedSize}, ${item.selectedColor}) - ${item.price.toFixed(2)}€`
  ).join('\n');

  const embed = {
    title: "🛍️ NEW ORDER - 24 WEARS",
    description: "⚠️ **PAYMENT PENDING** - Awaiting PayPal confirmation",
    color: 0xFFA500,
    fields: [
      {
        name: "📋 Order Number",
        value: orderData.orderId,
        inline: true
      },
      {
        name: "📅 Date",
        value: new Date().toLocaleString('de-DE'),
        inline: true
      },
      {
        name: "💳 Payment Status",
        value: "⏳ **PENDING**",
        inline: true
      },
      {
        name: "👤 Customer",
        value: orderData.customer.fullName,
        inline: false
      },
      {
        name: "📧 Email",
        value: orderData.customer.email,
        inline: false
      },
      {
        name: "📍 Delivery Address",
        value: `${orderData.customer.address.street}\n${orderData.customer.address.zip} ${orderData.customer.address.city}`,
        inline: false
      },
      {
        name: "🛒 Items",
        value: itemsList,
        inline: false
      },
      {
        name: "📦 Shipping",
        value: orderData.shipping === "0.00" ? "**FREE**" : `${orderData.shipping}€`,
        inline: true
      },
      {
        name: "💰 Total",
        value: `**${orderData.total}€**`,
        inline: false
      }
    ],
    footer: {
      text: "24 WEARS • Premium Streetwear"
    },
    timestamp: new Date().toISOString()
  };

  await fetch(webhookUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ embeds: [embed] })
  });
}

async function sendDiscordOrderConfirmation(data) {
  const webhookUrl = process.env.DISCORD_WEBHOOK_ORDERS;
  
  if (!webhookUrl) return;

  const embed = {
    title: "✅ ORDER CONFIRMED - PAYMENT RECEIVED",
    description: "🎉 **PAYMENT SUCCESSFUL** - Order ready for processing",
    color: 0x10b981,
    fields: [
      {
        name: "📋 Order Number",
        value: data.orderId,
        inline: true
      },
      {
        name: "🔐 PayPal Transaction ID",
        value: data.paypalTransactionId,
        inline: false
      },
      {
        name: "💳 Payment Status",
        value: "✅ **PAID**",
        inline: true
      }
    ],
    footer: {
      text: "24 WEARS • Premium Streetwear • ORDER CONFIRMED"
    },
    timestamp: new Date().toISOString()
  };

  await fetch(webhookUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ embeds: [embed] })
  });
}

async function sendDiscordNewsletterNotification(email) {
  const webhookUrl = process.env.DISCORD_WEBHOOK_NEWSLETTER;
  
  if (!webhookUrl) return;

  const embed = {
    title: "📧 NEW NEWSLETTER SUBSCRIPTION",
    color: 0xFFFFFF,
    fields: [
      {
        name: "Email Address",
        value: email,
        inline: false
      },
      {
        name: "Date & Time",
        value: new Date().toLocaleString('de-DE'),
        inline: false
      }
    ],
    footer: {
      text: "24 WEARS • Newsletter Subscription"
    },
    timestamp: new Date().toISOString()
  };

  await fetch(webhookUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ embeds: [embed] })
  });
}

async function sendCustomerEmail(orderData) {
  if (!process.env.EMAIL_USER) return;

  const mailOptions = {
    from: `"24 WEARS" <${process.env.EMAIL_USER}>`,
    to: orderData.customer?.email || 'customer@example.com',
    subject: `Order Confirmation - ${orderData.orderId}`,
    html: `
      <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
        <h1 style="color: #000; text-align: center;">24 WEARS</h1>
        <h2 style="color: #10b981;">✅ Payment Successful!</h2>
        
        <p>Thank you for your order!</p>
        
        <div style="background: #f3f4f6; padding: 20px; margin: 20px 0;">
          <h3>Order Details</h3>
          <p><strong>Order Number:</strong> ${orderData.orderId}</p>
          <p><strong>Transaction ID:</strong> ${orderData.paypalTransactionId}</p>
          <p><strong>Status:</strong> Paid ✅</p>
        </div>
        
        <p>We will process your order shortly and send you shipping information.</p>
        
        <p style="margin-top: 40px; color: #666; font-size: 12px;">
          24 WEARS - Premium Streetwear<br>
          Email: twenty.four.wears@gmx.de
        </p>
      </div>
    `
  };

  await transporter.sendMail(mailOptions);
}

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
  console.log(`Environment: ${process.env.NODE_ENV || 'development'}`);
});
