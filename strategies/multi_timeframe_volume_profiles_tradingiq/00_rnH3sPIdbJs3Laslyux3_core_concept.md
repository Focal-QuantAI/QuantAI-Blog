
# Core Concept

### 1. The Market Philosophy
This script operates on the principles of **Auction Market Theory**, a framework that views price movement as a continuous auction process seeking value. The core thesis is that markets gravitate towards areas of high liquidity and price acceptance, making it a sophisticated form of **Mean Reversion** strategy. By mapping volume distribution at specific price levels across multiple timeframes, the script aims to identify "fair value" zones (High Volume Nodes, or HVNs) and areas of rejection (Low Volume Nodes, or LVNs). The underlying principle is that price levels where significant trading occurred in the past will act as future support or resistance, as they represent consensus on value. The strategy seeks to generate alpha by identifying these structurally significant price zones, which are often defended or targeted by institutional participants.

### 2. The Trade Narrative
The script is designed to provide context, not signals. The ideal "story" it helps a trader identify is one of **price interacting with a historical value area**. For example, the narrative for a long entry might be: "After a sharp sell-off, the price is now entering the Value Area Low (VAL) of last week's profile, a level where significant buying previously occurred. We are looking for signs of absorption and a rotation back up towards the Point of Control (POC)." Conversely, a short setup would involve price approaching a prior High Volume Node or Value Area High (VAH) and showing signs of rejection. The script visualizes the battlefield before the trader commits capital, highlighting key zones of potential supply and demand.

### 3. Trigger Logic & Mechanics
This indicator is a decision-support tool; the trader is the ultimate trigger. Its power lies in the **confluence of multi-timeframe volume data**.

*   **Why these indicators?** The script doesn't combine disparate indicators; it layers the same powerful concept—the Volume Profile—from different timeframes (e.g., Daily, Weekly). This creates a hierarchical map of market structure. A confluence of a Daily POC with a Weekly VAL creates a high-conviction support zone that is far more robust than a level from a single timeframe.

*   **How do filters reduce noise?** The primary filter is the use of higher timeframes (HTFs). By focusing on volume structures built over days or weeks, the script filters out the noise of intraday fluctuations, improving the signal-to-noise ratio. Furthermore, it uses lower-timeframe data (e.g., 1-minute volume) to construct the HTF profiles, ensuring a high-resolution, accurate depiction of where business was actually conducted.

*   **The Catalyst:** The script flips from "observing" to providing an "actionable insight" when live price action intersects with a key level (POC, VAH, VAL) from one of the displayed HTF profiles. The inclusion of a Delta Profile option further refines this, revealing whether aggressive buyers or sellers controlled that level, adding a crucial layer of order flow context to the structural map.
    