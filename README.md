# SOLANA-DEFI-PROJECT
NEED HELP AND PARTNERS TO CREATE A DEFI PROJECT ON THE SOLANA NETWORK
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, Mint, TokenAccount, Transfer};

// Define the program ID
declare_id!("YourProgramID");

// Program module
#[program]
pub mod reward_token {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>, reward_rate: u8) -> Result<()> {
        let token_pool = &mut ctx.accounts.token_pool;
        token_pool.reward_rate = reward_rate; // e.g., 2%
        Ok(())
    }

    pub fn transfer_tokens(ctx: Context<TransferWithFee>, amount: u64) -> Result<()> {
        let reward_rate = ctx.accounts.token_pool.reward_rate as u64;
        let fee = (amount * reward_rate) / 100; // Calculate fee
        let transfer_amount = amount - fee;

        // Send fee to token pool
        let cpi_accounts = Transfer {
            from: ctx.accounts.from.to_account_info(),
            to: ctx.accounts.pool.to_account_info(),
            authority: ctx.accounts.authority.to_account_info(),
        };
        let cpi_context = CpiContext::new(ctx.accounts.token_program.to_account_info(), cpi_accounts);
        token::transfer(cpi_context, fee)?;

        // Proceed with normal token transfer
        let cpi_accounts_normal = Transfer {
            from: ctx.accounts.from.to_account_info(),
            to: ctx.accounts.to.to_account_info(),
            authority: ctx.accounts.authority.to_account_info(),
        };
        let cpi_context_normal = CpiContext::new(ctx.accounts.token_program.to_account_info(), cpi_accounts_normal);
        token::transfer(cpi_context_normal, transfer_amount)?;

        Ok(())
    }

    pub fn distribute_rewards(ctx: Context<DistributeRewards>) -> Result<()> {
        let token_accounts = &ctx.accounts.holders;
        let pool_balance = ctx.accounts.pool.amount;

        for holder in token_accounts.iter() {
            let reward = pool_balance * holder.amount / ctx.accounts.total_supply; // Pro-rate rewards
            // Transfer rewards to the holder
            let cpi_accounts = Transfer {
                from: ctx.accounts.pool.to_account_info(),
                to: holder.to_account_info(),
                authority: ctx.accounts.authority.to_account_info(),
            };
            let cpi_context = CpiContext::new(ctx.accounts.token_program.to_account_info(), cpi_accounts);
            token::transfer(cpi_context, reward)?;
        }

        Ok(())
    }
}

// Accounts required for the program
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init)]
    pub token_pool: ProgramAccount<'info, TokenPool>,
    pub authority: Signer<'info>,
}

#[derive(Accounts)]
pub struct TransferWithFee<'info> {
    pub from: Account<'info, TokenAccount>,
    pub to: Account<'info, TokenAccount>,
    #[account(mut)]
    pub pool: Account<'info, TokenAccount>, // Pool to collect fees
    pub authority: Signer<'info>,
    pub token_program: Program<'info, Token>,
}

#[derive(Accounts)]
pub struct DistributeRewards<'info> {
    pub holders: Vec<Account<'info, TokenAccount>>, // List of token holders
    #[account(mut)]
    pub pool: Account<'info, TokenAccount>,        // Pool from which rewards will be distributed
    pub authority: Signer<'info>,
    pub token_program: Program<'info, Token>,
    pub total_supply: u64,                        // Total token supply
}

#[account]
pub struct TokenPool {
    pub reward_rate: u8, // Reward rate as a percentage (e.g., 2%)
}
