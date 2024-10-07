<script>
import bootstrap from 'bootstrap/dist/js/bootstrap.bundle.min.js';
import { Wallet, TokenSendRequest } from 'mainnet-js';

export default {
  data() {
    return {
      isMonday: false,
      tooltipMessage: "Claiming is only available on Mondays (UTC+8)",
      satsDataMessage: 'Loading...',
      walletAddress: '',
      claimStatus: '',
      claimStatusColor: 'orange',
      config: {}, // Initialize config as an empty object,
      wallet: null,
      bchInUSD: 0,
      poolBCH: 0,
      poolHodl: 0
    };
  },
  mounted() {
    this.loadConfig(); // Load config when the component is mounted
    
    this.setClaimButtonStatus();
    this.fetchSatsData();

    // Initialize Bootstrap tooltips
    const tooltipTriggerList = [].slice.call(
      document.querySelectorAll('[v-b-tooltip]')
    );
    tooltipTriggerList.map(function (tooltipTriggerEl) {
      return new bootstrap.Tooltip(tooltipTriggerEl);
    });

    // Check every minute if the button should be enabled or disabled
    setInterval(this.setClaimButtonStatus, 60000);

    
  },
  methods: {
    async loadConfig() {
      try {            
        const response = await fetch('/config.json');
        this.config = await response.json();
        this.fetchSatsData(); // Call the function to fetch data after loading config
      } catch (error) {
        console.error('Error loading config:', error);
      }
    },
    async fetchSatsData() {
      // Ensure config is loaded before accessing it
      /* if (!this.config.apiBaseUrl) {
        console.error('Config not loaded. Unable to fetch sats data.');
        return;
      } */

      try {
        const response = await fetch(this.config.apiBaseUrl);
        const data = await response.json();

        if (data && data.active) {
          this.poolBCH = data.active.reduce((sum, item) => sum + item.sats, 0) / 100000000;
          this.poolHodl = data.active.reduce((sum, item) => sum + item.tokens, 0);
          this.bchInUSD = await this.getBCHPrice();
        
          const bchPoolUSD = (this.poolBCH * this.bchInUSD).toFixed(2);
          const hodlPoolUSD = (this.poolHodl * (this.poolBCH/this.poolHodl) * this.bchInUSD).toFixed(2);

          console.log("BCH in USD:" + this.bchInUSD);
          console.log("BCH Pool:" + this.poolBCH + "(" + bchPoolUSD + "usd)");
          console.log("HODL Pool:" + this.poolHodl + "(" + hodlPoolUSD + "usd)");
          console.log("Total:" + (Number(bchPoolUSD) + Number(hodlPoolUSD)) + "usd");

          const formattedAirdropTokens = this.formatAbbreviated((this.poolHodl * (10 / 100)) / 100000000);
          const formattedAirdropToken = this.formatAbbreviated((this.poolHodl * (10 / 100)) / 100 / 100000000);

          this.satsDataMessage = `Airdrop to claim: ${formattedAirdropTokens} 🚀\n
            First 100 to claim will get: ${formattedAirdropToken} 🚀 each`;
        } else {
          this.satsDataMessage = 'No sats data available.';
        }
      } catch (error) {
        this.satsDataMessage = 'Failed to load sats data.' + error;
      }
    },
    formatAbbreviated(number) {
      if (number >= 1e6) {
        return (number / 1e6).toFixed(1) + 'M'; // Millions
      } else if (number >= 1e3) {
        return (number / 1e3).toFixed(1) + 'K'; // Thousands
      } else {
        return number.toString(); // Smaller numbers
      }
    },
    isMondayUTC8() {
      const nowUTC = new Date();
      const nowUTC8 = new Date(nowUTC.getTime() + 8 * 60 * 60 * 1000);
      const day = nowUTC8.getUTCDay();

      return day === 2; // 1 represents Monday
    },
    setClaimButtonStatus() {
      this.isMonday = this.isMondayUTC8();
    },

    async getBCHPrice(){
      try {
        const response = await fetch(this.config.BCHAPIurl)
        const data = await response.json()
        const bchPrice = data['bitcoin-cash'].usd
        return bchPrice
      } catch (error) {
        console.error('Error fetching BCH price:', error)
        return null
      }
    },

    async claimTokens() {
      const walletAddress = this.walletAddress;
      const claimStatus = document.getElementById("claim-status");

      if (!this.config.enableClaim) {
        claimStatus.innerText = "Faucet is disabled.";
        claimStatus.style.color = "red";
        return;
      }


      // Check if it's Monday (UTC+8)
      if (!this.isMondayUTC8()) {
        claimStatus.innerText = "Claiming is only available on Mondays (UTC+8).";
        claimStatus.style.color = "red";
        return;
      }

      if (!walletAddress.startsWith("bitcoincash:z")) {
        claimStatus.innerText = "Invalid BCH address. Please use the correct format (bitcoincash:...)";
        claimStatus.style.color = "red";
        return;
      }

        
      const seedphase = import.meta.env.VITE_SEEDPHRASE;
      const derivationPathAddress = import.meta.env.VITE_DERIVATIONPATH;
      const tokenIdFungible = import.meta.env.VITE_TOKENID_FUNGIBLE;
      const wallet = await Wallet.fromSeed(seedphase, derivationPathAddress);

      const faucetWalletAddress = wallet.getDepositAddress();
      const balance = await wallet.getBalance();
      const tokenBalance = await wallet.getTokenBalance(tokenIdFungible);


      console.log(`faucetWalletAddress: ${faucetWalletAddress}`);
      console.log(`tokenBalance: ${tokenBalance}`);
      console.log(`BCH Balance: ${balance.bch} (${balance.bch * this.bchInUSD} usd)`);
       
      console.log("tokenBalance for eligibilty: ");
      const claimerWallet = await Wallet.watchOnly(walletAddress);
      const unformattedUserHodl = await claimerWallet.getTokenBalance(tokenIdFungible);
      const userHodl = Number(unformattedUserHodl / BigInt(100000000));
      
      console.log(typeof formatteduserHodl);
      console.log(typeof this.bchInUSD);

      console.log(`Claimer Hodl: ${userHodl} (${userHodl * this.bchInUSD}usd)`);

      if ((userHodl * this.bchInUSD) > 1){
        //Eligible
        this.claimStatus = "Claiming your tokens...";
        this.claimStatusColor = "orange";
        const claimToken = Math.floor(this.poolHodl * (10 / 100));//BigInt((this.poolHodl * (10 / 100)) / 100 / 100000000);
        console.log("Claimable Token: " + claimToken);
        console.log(typeof claimToken);
 
        const airdropOutput = new TokenSendRequest({
          cashaddr: walletAddress,
          tokenId: tokenIdFungible,
          amount: BigInt(claimToken)
        });
        const { txId } = await wallet.send([airdropOutput]);
        console.log(txId);
        

      }
      else {
        //Not Eligible

      }


      setTimeout(() => {
        this.claimStatus = "Tokens successfully claimed! Check your wallet for your tokens.";
        this.claimStatusColor = "green";
      }, 2000);
    },
  },
};

</script>

<template>
  <div id="app" class="container">
    <h3 class="text-center my-4">HODL (🚀) <i>Hold On for Dear Life</i></h3>
    
    <!-- Sats Data Section -->
    <div class="section sats-data card p-3 mb-4" id="sats-data">
      <div class="card-body text-center">
        <h6 class="card-title">Total Sats in Pool:</h6>
        <p class="card-text" id="sats-data-message">{{ satsDataMessage }}</p>
      </div>
    </div>

    <!-- Faucet Functionality Section -->
    <div class="section faucet-section card p-4 mb-4" id="faucet-section">
      <h6>Claim Your Hodl Tokens</h6>
      <p>Claim Hodl tokens by entering your BCH wallet address below:</p>
      <input
        type="text"
        id="wallet-address"
        placeholder="Enter your BCH address (bitcoincash:z...)"
        class="form-control mb-3"
        v-model="walletAddress"
      />

      <!-- Claim Button with Tooltip -->
      <button
        id="claim-button"
        @click="claimTokens"
        class="btn btn-warning btn-block mt-3"
        :disabled="!isMonday"
        v-b-tooltip.hover="tooltipMessage"
      >
        Claim Tokens
      </button>
      <p id="claim-status" :style="{ color: claimStatusColor }">{{ claimStatus }}</p>
    </div>

    <!-- New Swap Section -->
    <div class="section faucet-section card p-4 mb-4">
      <h2>Swap BCH to HODL</h2>
      <p>
        You can easily swap your BCH for Hodl tokens on Cauldron. Click the link below to start swapping:
      </p>
      <a        
        :href="config.swapTokenUrl"
        class="swap-link btn btn-primary btn-block"
        target="_blank"
      >
        Swap BCH to HODL
      </a>
    </div>

    <h1 class="text-center my-4">Tokenomics</h1> 

    <!-- Tokenomics Sections -->
    <div class="section card p-4 mb-4">
      <h2>1. Token Details</h2>
      <ul>
        <li><strong>Token ID:</strong> c9696fd9579b6703bd955a34a9c7b47fd06f71eaf158b7d5673cc60fbc0c6229</li>
        <li><strong>Token Name:</strong> Hodl</li>
        <li><strong>Symbol:</strong> 🚀</li>
        <li><strong>Total Supply:</strong> 1,000,000,000 (1 billion) Hodl</li>
        <li><strong>Decimals:</strong> 8</li>
        <li><strong>Description:</strong> Meme coin designed to engage and entertain crypto enthusiasts, with a focus on community-driven liquidity growth.</li>
      </ul>
    </div>

    <div class="section card p-4 mb-4">
      <h2>2. Distribution Categories</h2>
      <table class="table table-bordered">
        <thead>
          <tr>
            <th>Category</th>
            <th>Allocation</th>
            <th>Description</th>
          </tr>
        </thead>
        <tbody>
          <tr>
            <td>Liquidity</td>
            <td>60% (600,000,000 Hodl)</td>
            <td>Allocated for initial liquidity pools and future liquidity programs to ensure long-term token utility and market stability.</td>
          </tr>
          <tr>
            <td>Faucet Distribution</td>
            <td>30% (300,000,000 Hodl)</td>
            <td>Set aside for the weekly faucet claims to incentivize holding, liquidity contributions, and community engagement.</td>
          </tr>
          <tr>
            <td>Dev Fund</td>
            <td>10% (100,000,000 Hodl)</td>
            <td>Reserved for development, upgrades, and any future growth needs for the project.</td>
          </tr>
        </tbody>
      </table>
    </div>

    <div class="section card p-4 mb-4">
      <h2>3. Faucet Distribution Mechanism</h2>
      <h3>Weekly Claim Mechanism</h3>
      <ul>
        <li><strong>Faucet Reward:</strong> Based on the current <span class="highlight">airdrop pool</span> calculation.</li>
        <li><strong>Eligibility Criteria:</strong>
          <ul>
            <li>Users must hold at least <span class="highlight">$1 worth</span> of Hodl tokens.</li>
            <li>Each eligible user can only claim <span class="highlight">once per week</span>.</li>
            <li><span class="highlight">Only the first 100 eligible users</span> can claim their share from the airdrop pool each week.</li>
          </ul>
        </li>
        <li><strong>Liquidity Condition:</strong>
          <ul>
            <li>The faucet is <span class="highlight">only active</span> if liquidity in the Hodl pool increases by any amount compared to the previous week.</li>
            <li>The liquidity condition will be tracked via 
                <a        
                  :href="config.addLiquidityUrl"
                  target="_blank"
                >Cauldron 🚀's Pool</a>.
            </li>
            <li>This incentivizes the community to actively increase liquidity, making the faucet accessible.</li>
          </ul>
        </li>
      </ul>
    </div>

    <div class="section card p-4 mb-4">
      <h2>4. Liquidity Growth Focus</h2>
      <ul>
        <li><strong>Initial Liquidity:</strong> A small amount of liquidity will be provided by the developers to start.</li>
        <li><strong>Growth through Incentives:</strong> The combination of faucet rewards and the liquidity increase condition encourages holders to either add liquidity or encourage others to do so to activate the faucet.</li>
        <li>Community members are encouraged to provide liquidity on decentralized platforms like <span class="highlight">BCH’s Cashtokens</span>.</li>
      </ul>
    </div>

    <div class="section card p-4 mb-4">
      <h2>5. Token Utility</h2>
      <ul>
        <li><strong>Faucet Rewards:</strong> A direct, simple way for users to earn tokens by holding and participating in liquidity pools.</li>
        <li><strong>Liquidity Growth:</strong> By linking faucet activation to liquidity increase, Hodl builds a sustainable growth loop that benefits early adopters and the broader community.</li>
      </ul>
    </div>

    <div class="section card p-4 mb-4">
      <h2>6. Long-Term Vision</h2>
      <ul>
        <li><strong>Community-Driven:</strong> Over time, as more users contribute to liquidity and engage with the faucet, the token's circulating supply increases, and the community gains more control over the token’s growth and development.</li>
        <li><strong>Deflationary Potential:</strong> Future introduction of a small burn mechanism tied to transaction fees to reduce supply gradually, increasing the token’s scarcity.</li>
      </ul>
    </div>

    <div class="section card p-4 mb-4">
      <h2>Summary</h2>
      <p>
        The <span class="highlight">Hodl</span> token's design is simple yet engaging, with the primary focus on community-driven liquidity growth. By tying the faucet’s weekly rewards to increases in liquidity, you effectively create an incentive loop that encourages both holding and participation in liquidity pools. The distribution of <span class="highlight">60% liquidity, 30% faucet, and 10% for the development fund</span> ensures enough supply for growth, rewards, and project maintenance.
      </p>
    </div>
  </div>
</template>

<style>

body {
  font-family: 'Roboto', sans-serif;
  font-size: 11px; /* Slightly larger for readability */
  background-color: #f4f4f4;
  color: #333;
  line-height: 1.5; /* Adjusted line height for better readability */
  margin: 0;
  padding: 0;
}

.container {
  background: #fff;
  padding: 10px 20px; /* Reduced padding */
}

h1,
h2,
h3 {
  color: #ff6600;
  margin: 0.5rem 0; /* Reduced margin for headers */
}

table {
  width: 100%;
  border-collapse: collapse;
  margin-bottom: 10px; /* Reduced bottom margin */
}

table,
th,
td {
  border: 1px solid #ddd;
}

th,
td {
  padding: 8px; /* Reduced padding */
  text-align: left;
}

th {
  background-color: #ff6600;
  color: white;
}

ul {
  list-style-type: disc;
  padding-left: 15px; /* Reduced padding */
}

.section {
  margin-bottom: 20px; /* Reduced margin */
}

.highlight {
  color: #ff6600;
  font-weight: bold;
}

.swap-link {
  display: inline-block;
  background-color: #ff6600;
  color: white;
  padding: 8px 15px; /* Reduced padding */
  text-decoration: none;
  border-radius: 5px;
  font-weight: bold;
  margin: 15px 0; /* Reduced margin */
  text-align: center;
}

.swap-link:hover {
  background-color: #e65c00;
}

.sats-data,
.faucet-section {
  margin-top: 10px; /* Reduced margin */
  font-size: 12px; /* Adjusted font size */
  font-weight: bold;
  color: #ff6600;
  background-color: #fef8e0;
  border: 1px solid #ff6600; /* Reduced border thickness */
  border-radius: 8px; /* Slightly adjusted border radius */
  box-shadow: 0 3px 8px rgba(0, 0, 0, 0.1); /* Reduced shadow */
}

.sats-data h6,
.faucet-section h6 {
  color: #ff6600;
  margin: 0.5rem 0; /* Reduced margin for section titles */
}

.faucet-section input {
  padding: 8px; /* Reduced padding */
  width: 100%;
  max-width: 350px; /* Reduced max width */
  margin-bottom: 8px; /* Reduced margin */
  border: 1px solid #ddd;
  border-radius: 5px;
}

.faucet-section button {
  padding: 8px 15px; /* Reduced padding */
  background-color: #ff6600;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.faucet-section button:hover {
  background-color: #e65c00;
}

.highlight {
  color: #ffcc00; /* Highlight color */
  font-weight: bold;
}

@media (max-width: 576px) {
  h1 {
    font-size: 1.4rem; /* Adjust font size for mobile */
  }
  h2 {
    font-size: 1.15rem; /* Adjust font size for mobile */
  }
  h3 {
    font-size: 1rem; /* Adjust font size for mobile */
  }
  .btn {
    font-size: 0.9rem; /* Decrease button size for better touch targets */
  }
  .swap-link {
    margin-top: 0.5rem; /* Reduced space for link on mobile */
  }
}


/* Additional global styles can be added in assets/global.css */
@media (max-width: 576px) {
  /* Small devices (phones, up to 576px) */
  h1 {
    font-size: 1.4rem; /* Adjust font size for mobile */
  }
  h2 {
    font-size: 1.15rem; /* Adjust font size for mobile */
  }
  h3 {
    font-size: 1rem; /* Adjust font size for mobile */
  }
  .btn {
    font-size: 0.9rem; /* Decrease button size for better touch targets */
  }
  .swap-link {
    margin-top: 1rem; /* Add space for link on mobile */
  }
}

/*
header {
  line-height: 1.5;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
}

@media (min-width: 1024px) {
  header {
    display: flex;
    place-items: center;
    padding-right: calc(var(--section-gap) / 2);
  }

  .logo {
    margin: 0 2rem 0 0;
  }

  header .wrapper {
    display: flex;
    place-items: flex-start;
    flex-wrap: wrap;
  }
} */
</style>
